---
title: "Homelab Build-Out — Centralized Log Collection with WEF01, Two Ways"
author: uzrg
date: 2026-09-12 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Microsoft, HyperV, Windows Server]
tags: [Windows Event Forwarding, Elastic, Winlogbeat, Filebeat, Logstash, Sysmon, Syslog, IIS, DNS, How-To, Tutorial]
pin: false
mermaid: false
---

# A step-by-step guide: Windows Event Forwarding + a SIEM-bound pipeline, built two different ways

Every server in SUPERLAB up to this point has kept its own Event Logs,
its own IIS log files, its own DNS debug output — what's lacking is a
mechanism to centralize event logs into a single place to give some
overall visibility. Various SIEM products provide that capability, and
more, but how do the logs get there? This guide builds `WEF01`, a
single collector that every domain-joined Windows machine forwards its
events to natively, extended to also pull in Linux syslog and
file-based logs (IIS, DNS) that Windows Event Forwarding itself can't
handle.

Then it builds the same pipeline a second time, with a different set
of tools, specifically so you can compare them side by side. That
second build is the real point of this post: if you're handed either
"stand up centralized logging with Elastic Agent" or "stand up
centralized logging with Winlogbeat and Filebeat," you should be able
to follow this guide and end up with a working pipeline either way —
and understand why you might pick one over the other.

**Who this is for**: someone comfortable with Windows Server and Active
Directory basics, who hasn't necessarily built a Windows Event
Forwarding subscription or an Elastic Beats pipeline before.

## What you'll end up with

- `WEF01`, a Windows Event Collector every domain machine forwards to,
  scoped by three tiers (domain controllers, member servers,
  workstations) via GPO and AD group membership
- Sysmon deployed fleet-wide with MITRE ATT&CK-labeled telemetry riding
  those same subscriptions
- Linux syslog (and, separately, network-device syslog) landing on the
  same collector
- IIS and DNS logs — both file-based, neither supported by Windows
  Event Forwarding — here they are shipped through a custom built,
  near-real-time file-tailing pattern
- **Two parallel, independently working shipping pipelines** on WEF01
  itself: one built on Elastic Agent + Logstash, one built on
  Winlogbeat + Filebeat, both writing to local rotating files with a
  clearly marked spot to swap in Kafka once that exists
- A pipeline you've verified actually moves data, not just one where
  every service says "Running"

## Architecture

```
  Domain Controllers OU  --[GPO: WEF-Forwarding-DCs]-----------+
  Servers OU             --[GPO: WEF-Forwarding-MemberServers]-+
  Workstations OU        --[GPO: WEF-Forwarding-Workstations]--+
                                                                |
                                                                v
                                               WEF01 (ForwardedEvents channel)

  UBUNTU01 / future Linux, pfSense / network devices
      --[rsyslog / syslog]-->  WEF01

  IIS-hosting servers    --[FileSystemWatcher tail]-->  \\WEF01\LogDrop\IIS\<host>\...
  DC01 / DC02 DNS debug logs --[FileSystemWatcher tail]-->  \\WEF01\LogDrop\DNS\<host>\...

                              |
                              v
             WEF01's shipping layer, built two independent ways:

               Approach A:  Elastic Agent -> Logstash -> file output
               Approach B:  Winlogbeat ----------------> file output
                            Filebeat ------------------> file output

                              |
                              v
                    swap point, once it exists
                              |
                              v
                        Kafka (later)
```

## Prerequisites

- A Hyper-V host with an existing AD domain, at least one reachable DC
- A template VM you can clone Windows Server from (see this lab's own
  [generalized-template post]({% post_url 2026-07-13-phase1-plumbing-and-template %}) if you need one)
- An OU structure to place computer/group objects into (this lab uses
  `OU=Servers,OU=LabOU` — adjust to whatever yours is)
- A Linux host you can experiment with `rsyslog` on, if you want to
  follow the syslog section
- Set aside real time for this — building both pipelines end to end
  takes meaningfully longer than just one, given the number of gotchas
  documented along the way, and how long either takes depends a lot on
  your own pace and background

---

## Phase 1 — Provision the collector

Nothing special here beyond your normal VM build: clone from template,
join the domain, land the computer object in your servers OU (not the
default `CN=Computers` container — new AD objects should always go
somewhere deliberate, and it's worth remembering you can't link GPOs
to those built-in containers at all). Give it a second data disk
(`D:`) — you'll want somewhere other than the OS disk for log
output, the IIS/DNS drop share, and eventually Logstash's own install.

Sizing: 2 vCPU / 4 GB is enough to start. The JVM inside Logstash is
the single biggest memory hog in either pipeline; watch it if you add
volume later.

## Phase 2 — Sysmon, fleet-wide, with ATT&CK labels

Deploy Sysmon before building the subscriptions in Phase 3, so the
subscription queries can include its channel from day one instead of
being edited again later.

- **Config choice**: [Olaf Hartong's `sysmon-modular`](https://github.com/olafhartong/sysmon-modular)
  over the more common SwiftOnSecurity baseline, specifically because
  it embeds [MITRE ATT&CK](https://attack.mitre.org/) Tactic/Technique
  IDs directly into each rule's
  name — a simple field dissect downstream gets you structured
  `attack.tactic` / `attack.technique` fields for free. This lab used
  the repo's `sysmonconfig-with-filedelete.xml` variant for the added
  file-delete visibility. Pin the exact commit you pull the config
  from, same as you'd pin the Sysmon binary version — an unpinned
  `main`-branch fetch means every fleet-wide re-deploy risks pulling a
  config the rest of this guide wasn't written against:
  ```powershell
  Invoke-WebRequest "https://raw.githubusercontent.com/olafhartong/sysmon-modular/<commit-sha>/sysmonconfig-with-filedelete.xml" `
      -OutFile 'sysmonconfig.xml'
  Get-FileHash 'sysmonconfig.xml' -Algorithm SHA256   # record this alongside the commit SHA
  ```
  One thing to know before building that dissect: the subscriptions in
  Phase 3 use `<ContentFormat>RenderedText</ContentFormat>`, which
  delivers Sysmon's `RuleName` field embedded inside the rendered
  message text rather than as its own structured field — so the
  ATT&CK-label dissect is a text-parsing step against `message`, not a
  simple field reference. `RenderedEvents` (structured XML) is the
  alternative if you'd rather work with `RuleName` as a real field, at
  the cost of needing to parse that XML shape downstream instead.
- **Deployment mechanism**: whatever your lab already uses for
  software rollout (this lab used MECM, since that was already the
  established pattern). The two things that matter are (a) pin
  specific versions rather than tracking `latest`, and (b) roll out to
  workstations/member servers first, domain controllers as a separate,
  more deliberate pass later.

### The mistake worth knowing about before you make it yourself

Sysmon's real Event Log channel name is:

```
Microsoft-Windows-Sysmon/Operational
```

**Not** `Microsoft-Windows-Sysinternals-Sysmon/Operational` — that
longer name is an easy one to type from memory or carry over from
older documentation, and the failure mode is brutal: every
`Get-WinEvent -ListLog`, every subscription query, every health check
against the wrong name returns a generic "log not found" error that
looks exactly like Sysmon being broken, even when it's installed,
running, and logging perfectly. Verify the real name yourself before
trusting any script (including this one) that hardcodes it:

```powershell
wevtutil el | Select-String Sysmon
```

Second, smaller gotcha, once you have the right channel name: **Sysmon's
default manifest ACL does not grant `NT AUTHORITY\NETWORK SERVICE` read
access**, and that's the identity WinRM's Event Forwarding Plugin runs
as when a subscription pulls events. Every built-in Windows channel
(Security, Application, System) explicitly grants it; Sysmon's does
not. The symptom is a subscription that authorizes fine and forwards
every other channel, but silently drops Sysmon events — or fails with
a generic WSMan 1818 fault that looks identical to ordinary transient
churn. Diagnose it by diffing the raw ACL against a known-working
channel:

```powershell
wevtutil gl Microsoft-Windows-Sysmon/Operational | Select-String channelAccess
wevtutil gl Security | Select-String channelAccess
```

Security's SDDL includes `(A;;0x1;;;NS)`; Sysmon's does not include any
`;NS)` ACE at all. Fix it (idempotent, safe to re-run on a channel that
already has the grant):

```powershell
$channel = 'Microsoft-Windows-Sysmon/Operational'
$sddl = (wevtutil gl $channel | Select-String 'channelAccess').ToString() -replace 'channelAccess:\s*', ''
if ($sddl -notmatch ';NS\)') {
    # 0x1 is this build's grant, not a generic Windows access mask - Event Log
    # channel ACEs use their own scheme (1=Read, 2=Write, 4=Clear), unrelated
    # to file/registry/AD access masks. There's also no SACL segment to worry
    # about landing this in by mistake: unlike a full object SDDL, channelAccess
    # strings from wevtutil never carry an S: portion, so appending after the
    # last DACL ACE is always the correct, complete string.
    wevtutil sl $channel "/ca:$($sddl)(A;;0x1;;;NS)"
}
```

The general lesson under both of these: when something reads fine
locally as Administrator/SYSTEM but a service or subscription can't
consume it, look at what identity that service actually runs as, and
diff its permissions against something that already works. That
technique found this in minutes once applied; guessing at "maybe the
config is wrong" or "maybe reinstall it" cost a lot more time.

## Phase 3 — The collector role, three GPOs, three subscriptions

Three tiers map to three OUs — Domain Controllers, Servers, Workstations
— so build three independently-scoped subscriptions rather than one
broad one. Each can be paused, re-queried, or retired without touching
the others.

On WEF01:

```powershell
wecutil qc /quiet
winrm quickconfig -force
```

Then, per tier, a GPO linked to just that OU with:

- Computer Config → Admin Templates → Windows Components → Event
  Forwarding → **Configure target Subscription Manager**:
  `Server=http://WEF01.myhomelab.hv.lab:5985/wsman/SubscriptionManager/WEC,Refresh=900`
- Computer Config → Admin Templates → Windows Components → Windows
  Remote Management (WinRM) → WinRM Service → **Allow remote server
  management through WinRM**: Enabled

And a matching subscription created via `wecutil cs` with an XML
definition scoping `AllowedSourceDomainComputers` to that tier's AD
group (or the built-in `Domain Controllers` group for the DC tier).
Here's the real member-server subscription from this build, with the
group SID generalized — the DC and workstation subscriptions are the
same shape, just a different `Query` and a different group. Save this
as `WEF-MemberServers-template.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Subscription xmlns="http://schemas.microsoft.com/2006/03/windows/events/subscription">
  <SubscriptionId>WEF-MemberServers</SubscriptionId>
  <SubscriptionType>SourceInitiated</SubscriptionType>
  <Description>Member servers tier: System/Application errors, security logon, service-control, and Sysmon events.</Description>
  <Enabled>true</Enabled>
  <Uri>http://schemas.microsoft.com/wbem/wsman/1/windows/EventLog</Uri>
  <ConfigurationMode>Custom</ConfigurationMode>
  <Delivery Mode="Push">
    <Batching>
      <MaxItems>5</MaxItems>
      <MaxLatencyTime>60000</MaxLatencyTime>
    </Batching>
    <PushSettings>
      <Heartbeat Interval="900000"/>
    </PushSettings>
  </Delivery>
  <Query>
    <![CDATA[
      <QueryList>
        <Query Id="0">
          <Select Path="System">*[System[(Level=1 or Level=2) or (EventID=7034 or EventID=7040)]]</Select>
          <Select Path="Application">*[System[(Level=1 or Level=2)]]</Select>
          <Select Path="Security">*[System[(EventID=4624 or EventID=4625)]]</Select>
          <Select Path="Microsoft-Windows-Sysmon/Operational">*</Select>
        </Query>
      </QueryList>
    ]]>
  </Query>
  <ReadExistingEvents>false</ReadExistingEvents>
  <TransportName>HTTP</TransportName>
  <ContentFormat>RenderedText</ContentFormat>
  <Locale Language="en-US"/>
  <LogFile>ForwardedEvents</LogFile>
  <PublisherName>Microsoft-Windows-EventCollector</PublisherName>
  <AllowedSourceNonDomainComputers></AllowedSourceNonDomainComputers>
  <AllowedSourceDomainComputers>O:NSG:NSD:(A;;GA;;;$sid)</AllowedSourceDomainComputers>
</Subscription>
```

`$sid` there is a literal placeholder in the file — it doesn't resolve
itself. Pull the real SID from the AD group rather than typing one in
by hand (it changes if the group is ever recreated), then substitute
it into the template before saving the file `wecutil` will actually
read:

```powershell
$sid = (Get-ADGroup 'WEF-MemberServers').SID.Value
(Get-Content 'WEF-MemberServers-template.xml' -Raw) -replace '\$sid', $sid |
    Set-Content 'WEF-MemberServers.xml'
```

Create the subscription from the filled-in file:

```powershell
wecutil cs .\WEF-MemberServers.xml
```

To edit later without hand-crafting a diff, export the live
subscription, edit the XML, then re-import — this is also the fix for
the "stuck at the old query" symptom you'll hit if you try to edit a
subscription's query via `wecutil ss` and it doesn't seem to take:

```powershell
wecutil gs WEF-MemberServers /f:XML > WEF-MemberServers.xml   # export current state
# edit the file, then:
wecutil ds WEF-MemberServers   # delete
wecutil cs .\WEF-MemberServers.xml   # recreate
```

`wecutil gs .../f:XML` is also the safest way to build the *other* two
tiers' subscriptions in the first place: rather than hand-typing the
`AllowedSourceDomainComputers` SDDL and hoping the format is right (a
malformed SDDL string fails at `wecutil cs` with an error that won't
tell you which part is wrong), create your first subscription, export
it, and use that as the literal template for the next one — you know
the SDDL shape is correct because Windows itself just normalized it
into that exact string.

The three tiers' queries genuinely differ, not just cosmetically — the
DC tier pulls Kerberos/account-management-relevant events a member
server wouldn't generate meaningfully (`4672` privilege use, `4720`/
`4722`/`4724`/`4738` account changes, `4662` directory object access —
the [Windows Security Log Encyclopedia](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/default.aspx)
is a genuinely useful reference for what any Security event ID actually
means, rather than guessing from the number alone — plus the whole
`Directory Service` log and, once enabled, DNS's Analytical channel)
alongside the same base logon events; the workstation tier stays
deliberately narrow — logon events plus Sysmon — since endpoint
telemetry is what matters there, not account-management noise a
workstation rarely generates in the first place.

These queries are sized for a homelab, not a production fleet — a
real environment with thousands of member servers and workstations
will generate far more volume against even this "narrow" set than this
build ever sees. Expect to spend real time tuning each tier's query
against your own actual event volume before this scales past a
handful of hosts: narrowing `LogonType` values, excluding known-noisy
service accounts, or dropping an event class entirely once you've
confirmed nobody's actually
consuming it downstream.

![All three WEC subscriptions active, each with real source counts]({{ '/assets/img/gallery/wef01-wec-subscriptions-active.png' | relative_url }})
_All three tiers live on WEF01: 2 domain controllers, 18 member servers, 1 workstation, each forwarding to the same `ForwardedEvents` log_

### A subtle failure that looks like a permissions issue but isn't

If you scope a subscription to a **custom** AD security group instead
of individual computer SIDs, don't be surprised if newly-added members
get a flat "Access is denied" (WSMan fault 5) even though the group,
its membership, and AD replication all check out fine. This isn't a
group-authorization limitation — it's Kerberos. The forwarder
authenticates as its own computer account, and that account's TGT/PAC
(which is what encodes group membership for the SDDL check) is issued
at boot. A machine that was already running when it got added to the
group has a cached ticket that simply doesn't know about the new
membership yet.

**Fix**: reboot the machine after adding it to the forwarding group —
that's what actually gets a fresh TGT with the new group membership
baked in, not a service restart or a policy refresh — then force a
retry instead of waiting for the subscription's normal refresh
interval:

```powershell
Restart-Computer -Force
# once it's back up:
wecutil rs '<SubscriptionName>'
```

A separate, unrelated authorization layer needs widening too, or
non-admin source machines can't reach the collector at all regardless
of subscription scoping — the Event Forwarding Plugin's own WinRM-level
SDDL, by default, only allows `BUILTIN\Administrators` and `BUILTIN\Event
Log Readers`:

Fix it through the `WSMan:` provider rather than editing the registry
directly — the plugin's Security child key gets an auto-generated name
(`Security_<random>`), so discover it instead of hardcoding it:

```powershell
$resource = Get-ChildItem 'WSMan:\localhost\Plugin\Event Forwarding Plugin\Resources' |
    Select-Object -First 1
$security = Get-ChildItem $resource.PSPath | Where-Object PSChildName -like 'Security_*'
$sddlPath = Join-Path $security.PSPath 'Sddl'

$currentSddl = (Get-Item $sddlPath).Value
if ($currentSddl -notmatch ';;;AU\)') {
    $newSddl = $currentSddl -replace 'S:P', '(A;;GR;;;AU)S:P'
    Set-Item $sddlPath -Value $newSddl -Force
    Restart-Service WinRM -Force
}
```

The default out-of-the-box SDDL looks like
`O:NSG:BAD:P(A;;GA;;;BA)(A;;GR;;;ER)S:P(AU;FA;GA;;;WD)(AU;SA;GWGX;;;WD)`
— note there's no `AU` (Authenticated Users) ACE in the `D:` (DACL)
portion before `S:P` (the system audit ACL) begins. The fix above
inserts the missing ACE right before that boundary, which is where the
DACL always ends in this SDDL shape — it's a DACL insert, not a SACL
one: everything before `S:P` is still the DACL, `S:P` is only where the
*next* section (the SACL) begins.

Be clear about what this actually opens, though: `(A;;GR;;;AU)` is a
deliberate widening of an authorization boundary, not a narrow one.
Every authenticated domain principal can now reach the Event
Forwarding Plugin over WinRM — this grant has no concept of which
subscriptions or events a given machine should see, only whether it
can talk to the plugin at all. Subscription-level scoping
(`AllowedSourceDomainComputers` in Phase 3's XML) is the actual
control that limits which principals can forward which events; this
SDDL fix is a prerequisite for that scoping to matter, not a
replacement for it.

## Phase 4 — Linux and network-device syslog

No agent on the Linux side at all — the same "nothing extra runs on the
source" principle as everything else in this build. Point `rsyslog` at
WEF01:

```
# /etc/rsyslog.d/90-wef01.conf on the Linux host
*.* @wef01.myhomelab.hv.lab:514   # single @ = UDP, matching WEF01's udp input in Phase 6
```

This build's collector side listens on UDP specifically (Phase 6's
`udp` input, port 514) — rsyslog's `@@` prefix means TCP, so make sure
the sender side matches with a single `@` rather than defaulting to
the more common `@@` example you'll see in most rsyslog docs.

Network devices that support remote syslog natively (most firewalls,
switches) just need their syslog destination pointed at WEF01 the same
way — if your firewall is a shared, hands-off appliance in your
environment the way pfSense is in this lab, that's a change for
whoever owns it, not something to make unilaterally.

![Real UBUNTU01 syslog content landing in SyslogDrop]({{ '/assets/img/gallery/wef01-syslog-ubuntu01-content.png' | relative_url }})
_`syslog-1.log` on WEF01, systemd unit activity from UBUNTU01 — real content, not a connectivity test_

## Phase 5 — File-based logs: IIS and DNS, near-real time

Windows Event Forwarding can't handle IIS log files or DNS's debug text
log — they're not Event Log channels. Rather than polling for rotated
files every few minutes (real detection latency cost), this uses a
custom built `FileSystemWatcher`-based tailer that reacts to writes as
they happen:

- IIS opens its log files with share-read access, so the currently
  active file can be tailed while IIS is still writing to it.
- The tailer tracks a per-file byte offset in a small local state file
  (survives restarts) and appends only new bytes to
  `\\WEF01\LogDrop\IIS\<hostname>\` — **type first, hostname second**,
  not the other way around. This isn't arbitrary: it's what makes the
  Logstash tier-tagging filter in Phase 6 provably unambiguous
  regardless of what any given host is named (see that section for
  why hostname-first ordering is a real trap, not just a style
  choice). If a host runs more than one instance of this tailer — DC01
  here, tailing both IIS and DNS — each instance needs its own state
  file, not the shared default (see the real issue that caused, below).
- Run it as a Scheduled Task, "At startup," restart-on-failure.
- DNS's live Analytical channel turns out to be a dead end for
  forwarding — see the callout below — so DNS uses the classic debug
  text-file log (`Set-DnsServerDiagnostics -EnableLoggingToFile`)
  shipped through the exact same tailer/drop-share pattern as IIS.

The share needs a real ACL before any of this matters — everything
downstream treats whatever lands in `LogDrop` as legitimate, so a
share that's writable by more than it needs to be has no integrity
guarantee at all. Create the writer group first — same as
`WEF-MemberServers` and the other forwarding groups, it lands in
`OU=Groups,OU=LabOU`, not the default `CN=Users` container — then
membership, then the share and its ACL, all on WEF01:

```powershell
New-ADGroup -Name 'WEF-LogDrop-Writers' -GroupCategory Security -GroupScope DomainLocal `
    -Path 'OU=Groups,OU=LabOU,DC=myhomelab,DC=hv,DC=lab'

# every IIS/DNS host running Phase 5's tailer, by its computer account -
# DEVOPS01 and DC01/DC02 shown here as the running examples used
# elsewhere in this guide; add every other tailer host the same way
Add-ADGroupMember -Identity 'WEF-LogDrop-Writers' -Members 'DEVOPS01$', 'DC01$', 'DC02$'

New-Item -Path 'D:\LogDrop' -ItemType Directory -Force

# No -FullAccess for WEF01$ here - the shipper reads D:\LogDrop
# directly on WEF01, never over \\WEF01\LogDrop, so the machine
# account has no reason to hold share-level access at all. Grant the
# share owner explicitly instead of leaving every access parameter
# unset: New-SmbShare defaults to Everyone:Read on the share itself
# when none is given, which is a wider grant than either option here.
New-SmbShare -Name 'LogDrop' -Path 'D:\LogDrop' -FullAccess 'BUILTIN\Administrators'
Grant-SmbShareAccess -Name 'LogDrop' -AccountName 'WEF-LogDrop-Writers' -AccessRight Change -Force

# The NTFS ACL is the layer that actually governs local read access on
# WEF01 itself, and Get-Acl returns D:\'s inherited entries along with
# the folder's own - adding a rule on top of that leaves whatever was
# already inherited (BUILTIN\Users among it, on a default-formatted
# data volume) still in effect. Break inheritance explicitly instead
# of assuming the new rule is the only one that applies:
$acl = Get-Acl 'D:\LogDrop'
$acl.SetAccessRuleProtection($true, $false)   # protect + drop inherited ACEs
$acl.AddAccessRule((New-Object System.Security.AccessControl.FileSystemAccessRule(
    'WEF-LogDrop-Writers', 'Modify', 'ContainerInherit,ObjectInherit', 'None', 'Allow')))
# Breaking inheritance strips SYSTEM's access too, along with everything
# else - re-add it explicitly, or the shipper (running as SYSTEM on
# WEF01) silently loses read access to the files it's supposed to ship:
$acl.AddAccessRule((New-Object System.Security.AccessControl.FileSystemAccessRule(
    'NT AUTHORITY\SYSTEM', 'FullControl', 'ContainerInherit,ObjectInherit', 'None', 'Allow')))
$acl.AddAccessRule((New-Object System.Security.AccessControl.FileSystemAccessRule(
    'BUILTIN\Administrators', 'FullControl', 'ContainerInherit,ObjectInherit', 'None', 'Allow')))
Set-Acl 'D:\LogDrop' $acl
```

`WEF-LogDrop-Writers` holds **computer** accounts, not user accounts —
the Scheduled Task below runs the tailer as `SYSTEM`, so it's the
machine's own identity, not a logged-on user's, that actually reaches
the share over the network. Forgetting the trailing `$` when adding a
member is a common way to end up granting a nonexistent user object
instead of the real computer account. The shipper side (Elastic Agent
/ Winlogbeat / Filebeat) never touches the *share* at all — it reads
the already-landed files locally on WEF01, so it needs no SMB grant —
but it does need the NTFS `SYSTEM` grant above, since it's still
subject to the same local filesystem ACL as anything else reading
`D:\LogDrop` on the box.

Here's the actual tailer, unedited from this build — it runs unchanged
on every IIS host and on the DNS servers alike, just pointed at a
different source directory:

```powershell
# Start-LogTailer.ps1 - watches one or more directories and appends
# newly-written bytes to a drop-share, keyed by hostname.
param(
    # Semicolon-delimited, not a real [string[]] - a genuine array
    # parameter cannot survive a -File launch from Task Scheduler.
    # Task Scheduler always starts processes from a flat command-line
    # string (Win32 CreateProcess semantics), and PowerShell's -File
    # argument binding does not reconstruct an array from either
    # space-separated or comma-separated tokens on that flattened line
    # (both were tried and silently bound only the first value) - only
    # a single delimited string round-trips intact.
    [Parameter(Mandatory)]
    [string]$LogDirectoriesRaw,

    [Parameter(Mandatory)]
    [string]$DropShare,

    # Defaults to living next to the script itself rather than a
    # hardcoded lab-specific path - this script has no dependency on
    # any particular directory convention, so nothing here should
    # assume one either. Override explicitly if you deploy the script
    # somewhere read-only.
    [string]$StateFilePath = (Join-Path $PSScriptRoot 'tailer-state.json')
)

$LogDirectories = $LogDirectoriesRaw -split ';'
$stateFile = $StateFilePath
$sweepIntervalSeconds = 15

# Shared across the main thread and event-handler threads, so both the
# Changed-event handler and the periodic sweep read/write the same
# offsets without racing each other.
$sync = [hashtable]::Synchronized(@{})
if (Test-Path $stateFile) {
    try {
        (Get-Content -Raw $stateFile | ConvertFrom-Json).PSObject.Properties |
            ForEach-Object { $sync[$_.Name] = [int64]$_.Value }
    } catch {
        # A state file can end up truncated if the process was killed
        # mid-write. Starting clean (re-ingesting from offset 0 on next
        # write) beats a scheduled task that crash-loops forever because
        # startup itself throws on a JSON parse.
        Write-Warning "State file $stateFile is unreadable, starting clean: $($_.Exception.Message)"
    }
}

function Copy-NewBytes {
    param([string]$SourcePath, [string]$DropShare, [hashtable]$Sync, [string]$StateFile)

    if (-not (Test-Path $SourcePath)) { return }

    $file = Get-Item $SourcePath
    $lastOffset = if ($Sync.ContainsKey($SourcePath)) { [int64]$Sync[$SourcePath] } else { 0 }

    # File shrank (rotated/replaced) - start over from the top.
    if ($file.Length -lt $lastOffset) { $lastOffset = 0 }
    if ($file.Length -eq $lastOffset) { return }

    try {
        # For an IIS source (...\LogFiles\W3SVC1\u_ex....log) this
        # correctly yields the site ID, W3SVC1. For DNS's debug log
        # (C:\Windows\System32\dns\dns.log) it yields "dns" - a
        # redundant-looking extra "dns" segment in the drop-share path
        # (LogDrop\DNS\<host>\dns\...), not an issue: Filebeat's glob
        # matches on the IIS/DNS segment one level up in the drop-share
        # path, not on this folder name, so the duplicate "dns" here
        # doesn't break anything downstream.
        $siteFolder = Split-Path (Split-Path $SourcePath -Parent) -Leaf
        $destPath = Join-Path $DropShare "$siteFolder\$($file.Name)"
        $destDir = Split-Path $destPath -Parent
        if (-not (Test-Path $destDir)) { New-Item -Path $destDir -ItemType Directory -Force | Out-Null }

        $srcStream = [System.IO.File]::Open($SourcePath, 'Open', 'Read', 'ReadWrite')
        $srcStream.Seek($lastOffset, 'Begin') | Out-Null
        $newLength = $file.Length - $lastOffset
        $buffer = New-Object byte[] $newLength
        $srcStream.Read($buffer, 0, $newLength) | Out-Null
        $srcStream.Close()

        $destStream = [System.IO.File]::Open($destPath, 'Append', 'Write', 'Read')
        $destStream.Write($buffer, 0, $buffer.Length)
        $destStream.Close()

        $Sync[$SourcePath] = $file.Length
        # Write-then-rename rather than writing $StateFile directly - the
        # Changed-event handler and the periodic sweep can both reach this
        # line close together, and a torn write from two overlapping
        # writers is a worse failure than one of them losing this pass's
        # update (which the next pass corrects anyway).
        $tempFile = "$StateFile.tmp"
        ($Sync | ConvertTo-Json) | Set-Content -Path $tempFile -Encoding utf8
        Move-Item -Path $tempFile -Destination $StateFile -Force
    } catch {
        Write-Warning "Skipped $SourcePath this pass: $($_.Exception.Message)"
    }
}

$watchers = foreach ($dir in $LogDirectories) {
    $w = New-Object System.IO.FileSystemWatcher($dir, '*.log')
    $w.IncludeSubdirectories = $false
    $w.NotifyFilter = [System.IO.NotifyFilters]'LastWrite,FileName,Size'
    $w.EnableRaisingEvents = $true

    # NOTE: the Register-ObjectEvent action block runs in the same
    # runspace as the script that registered it - script-scope
    # variables and functions are visible here directly. $using: does
    # NOT apply in this context (it's only meaningful for
    # remoting/ForEach-Object -Parallel) and throws on every event if
    # used here - a real issue hit during this build, caught only
    # because nothing was being copied despite the process staying
    # alive with no visible error.
    Register-ObjectEvent -InputObject $w -EventName Changed -Action {
        Copy-NewBytes -SourcePath $Event.SourceEventArgs.FullPath `
            -DropShare $DropShare -Sync $sync -StateFile $stateFile
    } | Out-Null

    $w
}

Write-Host "Tailing $($LogDirectories -join ', ') -> $DropShare"

# Safety-net sweep: catches anything the Changed event missed or
# coalesced under heavy write load - FileSystemWatcher is not 100%
# reliable under load, so relying on it alone risks silently losing data.
while ($true) {
    Start-Sleep -Seconds $sweepIntervalSeconds
    foreach ($dir in $LogDirectories) {
        Get-ChildItem -Path $dir -Filter '*.log' -ErrorAction SilentlyContinue | ForEach-Object {
            Copy-NewBytes -SourcePath $_.FullName -DropShare $DropShare -Sync $sync -StateFile $stateFile
        }
    }
}
```

Register it as an "At startup" Scheduled Task so it survives reboots
and restarts on its own if it ever crashes:

```powershell
$action = New-ScheduledTaskAction -Execute 'powershell.exe' -Argument `
    '-NoProfile -ExecutionPolicy Bypass -File C:\LabOps\Start-LogTailer.ps1 -LogDirectoriesRaw "C:\inetpub\logs\LogFiles\W3SVC1" -DropShare "\\WEF01\LogDrop\IIS\DEVOPS01"'
$trigger = New-ScheduledTaskTrigger -AtStartup
$settings = New-ScheduledTaskSettingsSet -RestartCount 999 -RestartInterval (New-TimeSpan -Minutes 1)
Register-ScheduledTask -TaskName 'LogTailer' -Action $action -Trigger $trigger `
    -Settings $settings -User 'SYSTEM' -RunLevel Highest
```

**A real issue this exact script hit, found on the one host running two
instances of it**: DC01 tails both its own IIS logs and DNS's debug
log, as two separate scheduled tasks running the same script content
under different names. Both instances defaulted to the same state
file, since the original version of this script hardcoded
`C:\LabOps\iis-tailer-state.json` rather than deriving it per-instance
— two independent processes writing to the same JSON file on every
flush, racing each other and silently corrupting whichever one wrote
last. `-StateFilePath` above exists specifically to fix this: give
each instance running on the same host its own explicit, distinct
path (e.g. `iis-tailer-state.json` vs `dns-tailer-state.json`) rather
than relying on the default. A host running only one instance of this
script never hits this — it's specific to any host, like DC01 here,
that has more than one reason to run it.

Both sources landing in the type-first layout, viewed locally on WEF01
— `LogDrop\IIS\<host>\` and `LogDrop\DNS\<host>\`, exactly as Phase 6's
tier-tagging filter depends on:

![IIS logs landing in LogDrop\IIS\DC01\W3SVC1]({{ '/assets/img/gallery/wef01-logdrop-iis-typefirst.png' | relative_url }})
_DC01's own IIS logs, tailed and landed under the type-first path_

![DNS debug logs landing in LogDrop\DNS\DC01\dns]({{ '/assets/img/gallery/wef01-logdrop-dns-typefirst.png' | relative_url }})
_DC01's DNS debug log, same pattern — note the harmless extra `dns` segment from the site-folder derivation_

### Why DNS can't just forward its Analytical channel

The DNS Server's Analytical channel is a real ETW-backed Windows Event
Log channel, which makes "just forward it like everything else" the
obvious first idea — and it doesn't work. Analytic/Debug channels are a
"direct channel" type that Windows restricts hard: **no external
consumer can query or subscribe to one while it's enabled**, full stop.
This isn't a permissions gap you can fix with an ACL — it fails the
same way whether you try `Get-WinEvent`/`wevtutil` (`EvtQuery`) or a
real-time `.NET EventLogWatcher` (`EvtSubscribe`), both with the exact
same "cannot be performed over an enabled direct channel, must first be
disabled" error. The MMC Event Viewer *looks* like it reads these
channels live, but it has special-cased GUI logic — disable, snapshot,
re-enable, transparently — that isn't exposed to any external consumer,
including WEC. If you hit this with any Analytic/Debug channel, stop
looking for a forwarding fix and reach for the classic debug-log
workaround instead.

## Sizing and retention, with real numbers from this build

Phase 1 suggested 2 vCPU / 4 GB as a starting point. Double-check that
the VM actually got that — this lab's WEF01 ended up provisioned with
only about 3.2 GB total, not the full 4 GB, and nobody caught the gap
until it mattered: it's a real contributor to the Logstash heap crash
covered below, once WEF01 picked up its own MECM/SCOM monitoring
agents on top of everything else already running on it.

Size it for what you're actually running, and verify the allocation
rather than trust the request:
- **Single pipeline** (the realistic production deployment): 4 GB is
  fine. Confirm it — don't take it on faith the way this build did.
- **Both pipelines concurrently**, as in this comparison build: budget
  6–8 GB. The box is also a monitored endpoint generating its own
  Sysmon/agent telemetry on top of everything it's ingesting from the
  rest of the fleet, and that overhead is not optional.

**Disk**: `D:` is 60 GB in this build — Logstash's install, both
pipelines' rotating output, and the IIS/DNS drop-share all share it.
The two output styles behave very differently, and only one of them is
actually bounded:

- **Logstash's file output started out unbounded, and that genuinely
  caught up with this build.** `wef-events-<date>.log` rotated daily,
  not by size, and nothing capped how large a single day's file got in
  between. An early busy day hit 8 GB before rolling over — noted at
  the time as "a data point, not a ceiling." A later day proved that
  right the hard way: **34.7 GB in one file**, on a 60 GB disk, free
  space down to under 9 GB before anyone caught it. The fix, deployed
  and verified on this exact build:
  - **Hourly buckets, not daily.** The `file` output plugin has no
    native size-based rotation — only date-pattern path rotation is
    real (confirmed against the current plugin docs, not assumed).
    `wef-events-%{+YYYY-MM-dd-HH}.log` bounds the worst case to about
    an hour of volume per file instead of a full day.
  - **An adaptive cleanup Scheduled Task**, hourly, purging files past
    a retention window that tightens as free space drops — 24h
    normally, 12h under 20 GB free, 6h under 10 GB — rather than one
    fixed window sized for an average day that a bad day blows through.
    Running it once reclaimed the stale 34.7 GB file immediately:
    8.4 GB free went to 56 GB in seconds.
  - The drop-share needs the same kind of cleanup task regardless (see
    below) — this is that same maintenance pattern applied a second
    time, not new work.
- **The Beats outputs are bounded.** `rotate_every_kb: 102400` ×
  `number_of_files: 10` caps each Beat's retained history at roughly
  **1 GB**, oldest file dropped automatically as new ones roll in.

Don't take "bounded" as "sized correctly for your volume," though —
1 GB of retained history and 24h of hourly Logstash buckets are both
just this build's starting point. Watch real free space, not just
whether a cap exists at all.

The cleanup script, unedited from this build — the retention window
tightens itself instead of assuming every day looks like an average
one:

```powershell
# Cleanup-LogstashOut.ps1 - bounds D:\LogstashOut total size.
# Hourly output files bound the size of any ONE file; this bounds how
# many of them accumulate. Retention tightens automatically if free
# space is already under pressure, rather than a single fixed window
# that assumes today's volume looks like every other day's.
$logstashOutPath = 'D:\LogstashOut'
$freeGB = (Get-PSDrive D).Free / 1GB
$retentionHours = if ($freeGB -lt 10) { 6 } elseif ($freeGB -lt 20) { 12 } else { 24 }
Get-ChildItem -Path $logstashOutPath -File -Filter 'wef-events-*.log' -ErrorAction SilentlyContinue |
    Where-Object LastWriteTime -lt (Get-Date).AddHours(-$retentionHours) |
    Remove-Item -Force
```

Registered as an hourly Scheduled Task the same way the drop-share
cleanup task is (below) — `-Once -At (Get-Date)` with an hourly
`-RepetitionInterval` is the pattern for "run on a recurring schedule
starting now," not the one-shot it looks like at a glance:

```powershell
$action = New-ScheduledTaskAction -Execute 'powershell.exe' -Argument `
    '-NoProfile -ExecutionPolicy Bypass -File D:\Logstash\Cleanup-LogstashOut.ps1'
$trigger = New-ScheduledTaskTrigger -Once -At (Get-Date) `
    -RepetitionInterval (New-TimeSpan -Hours 1) -RepetitionDuration (New-TimeSpan -Days 3650)
Register-ScheduledTask -TaskName 'WEF-LogstashOut-Cleanup' -Action $action -Trigger $trigger `
    -User 'NT AUTHORITY\SYSTEM' -RunLevel Highest
```

`-RepetitionDuration ([TimeSpan]::MaxValue)` looks like the obvious
choice for "run forever" and fails outright — `Register-ScheduledTask`
rejects it with "The task XML contains a value which is incorrectly
formatted or out of range," because the underlying Task Scheduler XML
schema can't represent a duration that large. A long-but-bounded span
like 3650 days works fine and means the same thing in practice.

**Logstash's JVM heap — a real incident, not a hypothetical.** This
build's `jvm.options` shipped with Logstash's own stock `-Xms1g`/
`-Xmx1g` defaults untouched — nobody had a reason to raise them yet.
Stock defaults ran out for real:
`java.lang.OutOfMemoryError: Java heap space`, both pipeline worker
threads dead, Logstash down — triggered once WEF01's own Sysmon volume
spiked after the MECM/SCOM agents landed on the box itself (see "What's
next"). A monitored endpoint generates its own telemetry on top of
whatever it's collecting from the rest of the fleet, and 1 GB wasn't
enough headroom for both loads at once.

Two things worth taking from this, not just the number:

- **Don't treat any heap-sizing recommendation — including this
  post's own — as fixed.** Watch actual JVM memory under real load and
  raise it before you hit the wall, not after. Logstash's own
  monitoring API gives you the real numbers without guessing from
  `Get-Process`:
  ```powershell
  (Invoke-RestMethod http://localhost:9600/_node/stats/jvm).jvm.mem.heap_used_percent
  ```
  Trending that over time — not just checking it once — is what would
  have shown this build's heap climbing toward the wall before it hit
  it, rather than finding out from a `FATAL` line after the fact.
- **The crash was silent.** `Get-Process java` kept showing a live
  process the entire time Logstash was down; only the log's `FATAL`
  entries said anything was wrong. Same lesson this build keeps
  repeating: a process existing is not the same claim as a process
  working.

Set the heap directly in `config/jvm.options`. A separate
`jvm.options.d/heap.options` file did **not** get picked up in this
Logstash build — verify your version actually reads that directory
before relying on it, and confirm against the running process's
command line, not the file you wrote. Append these two lines at the
**end** of the existing file rather than editing near the top —
`jvm.options` ships with a long list of other flags and commented-out
defaults, and there's no reason to disturb any of them just to add a
heap override:

```
-Xms1536m
-Xmx1536m
```

1536 MB covers this pipeline's real volume, Sysmon spike included.
Raise it further if you add enrichment filters (see Enrichment below)
or take on higher-volume sources. Confirm the value actually took
effect against the live process — not the config file:

```powershell
(Get-CimInstance Win32_Process -Filter "Name='java.exe'").CommandLine
```

Validate any config change before restarting the real service. This
alone would have caught the parser-breaking regex above in seconds,
instead of a live crash-and-restart cycle:

```powershell
& 'D:\Logstash\bin\logstash.bat' -f 'D:\Logstash\config\wef01-pipeline.conf' --config.test_and_exit
```

**The `ForwardedEvents` channel itself has a size cap, and it's not
generous.** `wevtutil gl ForwardedEvents` on this build's own WEF01
shows `maxSize: 20971520` — 20 MB, `retention: false` — the default,
never touched by anything in this guide. Every event from the whole
fleet lands in that one channel before Elastic Agent or Winlogbeat
ever reads it, and at fleet-wide volume 20 MB wraps fast; once it
does, `retention: false` means old events are gone, not archived. Size
it up before that matters, not after:

```powershell
wevtutil sl ForwardedEvents /ms:1073741824   # 1 GB
```

The right number scales with fleet size and how long a shipper outage
should be survivable without losing events — a shipper that's down for
an hour needs the channel to hold at least an hour's worth of the
fleet's real forwarding volume, not just look big on paper.

**DNS debug logging volume.** Phase 5's workaround — classic
`Set-DnsServerDiagnostics -EnableLoggingToFile` — is verbose by
design: every query gets a line, not just the interesting ones, and a
busy resolver produces multiple gigabytes a day of it. Treat it as
what it is — debug-level logging you'd never leave running on a
production DNS server outside a troubleshooting window. It's fine here
because DC01/DC02 aren't under real query load. On a busier resolver,
don't just flip it on the same way — scope it down first: sample
instead of capturing everything, shorten retention, or restrict to
specific event categories via the diagnostics cmdlet's other switches.

**Drop-share retention.** The tailer only ever appends — it never
deletes anything from `\\WEF01\LogDrop\`, and it's not meant to. Pair
it with a separate, dedicated cleanup Scheduled Task that purges
anything older than a conservative window. 24–48h is plenty: what
actually prevents re-ingestion is Elastic Agent/Filebeat's own
read-offset tracking, not how long the drop-share copy sticks around,
so don't over-retain it out of caution.

```powershell
Get-ChildItem 'D:\LogDrop' -Recurse -File |
    Where-Object LastWriteTime -lt (Get-Date).AddHours(-48) |
    Remove-Item -Force
```

## Firewall rules, all in one place

Every port this build needs open, in one place instead of scattered
across phases:

**TCP 5985, inbound on WEF01**: WinRM — subscription manager plus
event push from every forwarding source.

**UDP 514, inbound on WEF01**: syslog from Linux and network devices
(Phase 4). Only UDP is used anywhere in this build — Phase 4's sender
config and Phase 6's collector input are both UDP. Open TCP 514 too
only if a specific sender actually needs it; don't open it by default.

**UDP 5514, inbound on WEF01**: Filebeat's demo syslog listener
(approach B only). Skip it if you're not running the side-by-side
comparison.

**TCP 5044, loopback only on WEF01**: Beats to Logstash — never leaves
the host.

**TCP 445, inbound on WEF01**: SMB, for the `\\WEF01\LogDrop\` share —
every IIS and DNS host running Phase 5's tailer writes here. Easy to
overlook because nothing else in this build touches SMB, but the
tailer is silently dead without it: `File and Printer Sharing (SMB-In)`
isn't reliably on by default on every profile, and a fresh Windows
install won't have it enabled just because you created a share.

That's it — nothing else needs a rule anywhere. Every source machine
only makes outbound connections (pushing events over WinRM, forwarding
syslog, writing to the drop-share over SMB), so DC01/DC02, the member
servers, and the workstations need nothing opened at all. Outbound
5985 and outbound 445 are what actually carry that traffic from each
source, and both are allowed by default on a stock Windows firewall —
but if yours is locked down past the out-of-the-box profile, verify
both explicitly. Inbound being covered on WEF01's side proves nothing
about outbound elsewhere.

One thing to know if WinRM has never been touched on WEF01 before this
build: `winrm quickconfig -force` (Phase 3) is what binds the WinRM
listener to the network interface in the first place. A fresh Windows
install has the WinRM *service* running, but no listener configured
at all — not even a loopback one — until `quickconfig` — or the
equivalent GPO-driven listener creation — opens a real HTTP listener
and the matching firewall rule. Run `wecutil qc` without `winrm
quickconfig` and you'll get a subscription that looks correctly
configured with no listener for anything to actually reach.

## Production hardening (out of scope here, but worth knowing)

Everything here is unencrypted. That's an acceptable trade-off in an
isolated lab; it is not one to carry into anything internet-facing or
handling real user data:

- **WinRM** runs over plain HTTP (5985). Production wants HTTPS (5986)
  with a real certificate — which also means reworking the
  SubscriptionManager GPO value and the WinRM listener config to match.
- **Syslog** over UDP 514 is unencrypted and unauthenticated. Anyone
  who can reach the port can inject events. Close that gap with
  TLS-wrapped syslog (RFC 5425, which runs over TCP, not UDP) or an
  IPsec-protected segment.
- **The Beats protocol** between Elastic Agent/Winlogbeat/Filebeat and
  Logstash is loopback-only here, which sidesteps the problem entirely.
  The moment Logstash lives on a different host than its shippers,
  that link needs TLS (`ssl_enabled` on both the beats input and each
  shipper's output) — not a plaintext hop across the network.
- **The `LogDrop` share** carries IIS and DNS log content — hostnames,
  URLs, query names — over plain SMB. Without SMB signing (mandatory)
  and SMB encryption enabled on the share, that traffic is readable to
  anyone who can see the wire between an IIS/DNS host and WEF01.

None of this blocks anything in this guide. It's the list to work
through before this pattern leaves a lab.

## Phase 6 — Shipping layer, approach A: Elastic Agent + Logstash

Install Elastic Agent in **standalone mode** (no Fleet/Kibana
dependency) plus Logstash on WEF01 itself. Four inputs, one output:

```yaml
outputs:
  default:
    type: logstash
    hosts: ["127.0.0.1:5044"]

inputs:
  - type: winlog
    id: windows-forwarded-events
    streams:
      - data_stream: { dataset: windows.forwarded }
        name: ForwardedEvents
        forwarded: true

  - type: udp
    id: syslog-udp
    streams:
      - data_stream: { dataset: udp.syslog }
        host: "0.0.0.0:514"
        processors:
          - syslog: { field: message, format: auto }

  - type: filestream
    id: iis-logdrop
    streams:
      - data_stream: { dataset: iis.access }
        paths: ['D:\LogDrop\IIS\*\*\*.log']

  - type: filestream
    id: dns-logdrop
    streams:
      - data_stream: { dataset: dns.debug }
        paths: ['D:\LogDrop\DNS\*\*\*.log']
```

Elastic Agent installs as a Windows service (`Elastic Agent`), binary
and config landing at
`C:\Program Files\Elastic\Agent\elastic-agent.yml`. Standalone mode
skips the "enrollment" step Fleet-managed agents require entirely:
drop the config in place, (re)start the service, and it runs the
inputs defined in the file immediately. `elastic-agent.exe status`
should report `(HEALTHY) Running` with `fleet (STOPPED, Not
enrolled)` — that's the expected, correct state for this mode, not an
error to chase.

Logstash's own pipeline stays deliberately boring: `beats` input on
5044, `file` output with size-based rotation. That output stanza is
the one thing that changes when Kafka exists later — nothing upstream
of it needs to know or care. One real gotcha in how it's launched: this
build's scheduled task runs Logstash with
`-f D:\Logstash\config\wef01-pipeline.conf` directly, and passing `-f`
on the command line makes Logstash **ignore `pipelines.yml`
entirely** (it logs `Ignoring the 'pipelines.yml' file because
modules or command line options are specified`). If you're used to
multi-pipeline setups via `pipelines.yml`, know that a single `-f`
flag overrides that whole mechanism instead of adding to it.

```
input {
  beats {
    port => 5044
  }
}

filter {
  # Tier tagging: which collection path this event arrived through,
  # so downstream filtering by criticality doesn't require re-deriving
  # it from the hostname or dataset later.
  #
  # The drop-share layout is type-first: \\WEF01\LogDrop\IIS\<host>\...
  # and \\WEF01\LogDrop\DNS\<host>\... - NOT <host>\IIS or <host>\DNS.
  # That ordering is what makes the match below provably unambiguous:
  # "LogDrop\IIS" can only ever be that fixed literal boundary, never
  # a coincidence of some host's own name, because nothing (least of
  # all a hostname) can appear between "LogDrop" and the type segment
  # that immediately follows it. A bare /IIS/ or /DNS/ substring match
  # against a host-first layout doesn't have that guarantee - see the
  # callout below for the real, live issue that came from exactly this.
  if [event][dataset] == "windows.forwarded" {
    mutate { add_field => { "[wef][tier]" => "windows_event_forwarding" } }
  } else if [event][dataset] == "udp.syslog" {
    # Match Phase 6's own udp input exactly (data_stream.dataset:
    # udp.syslog) rather than a /^syslog/ regex - that regex looks
    # plausible but never actually matches this build's real dataset
    # value, which starts with "udp.", not "syslog". An exact match
    # here also removes the earlier fallback on [log][source][address]
    # being merely truthy, which had no way to rule out a non-syslog
    # event that happened to carry an address field of its own.
    mutate { add_field => { "[wef][tier]" => "syslog" } }
  } else if [log][file][path] =~ /LogDrop\\IIS/ {
    mutate { add_field => { "[wef][tier]" => "iis" } }
  } else if [log][file][path] =~ /LogDrop\\DNS/ {
    mutate { add_field => { "[wef][tier]" => "dns" } }
  }
}

output {
  file {
    # Hourly, not daily - see Sizing and retention for why a single
    # day's file reaching 34.7 GB made this a real fix, not a nice-to-have.
    path => "D:/LogstashOut/wef-events-%{+YYYY-MM-dd-HH}.log"
    codec => json_lines
  }
}
```

Verify a filter like this against a real event before trusting it.
Don't assume a field name matches what a plugin's docs say — check. A
one-line `stdout { codec => rubydebug }` output, alongside or instead
of the file output, shows the exact field structure Logstash is
actually working with in a few seconds:

```
output {
  stdout { codec => rubydebug }
}
```

A genuinely healthy Windows-forwarded event's `rubydebug` output looks
like the excerpt below. `event.dataset` — not `data_stream.dataset` —
is where `windows.forwarded` actually lives, which is exactly what the
filter's first condition checks:

```ruby
{
    "event" => {
        "dataset" => "windows.forwarded",
        ...
    },
    "data_stream" => {
        "dataset" => "windows.forwarded",
        ...
    },
    ...
}
```

(This excerpt is literal, not illustrative — copied from a real event
this build's `winlog` input produced, both fields carrying the
identical unnamespaced value shown above. If you've worked with ECS
data streams before, that should look surprising: `data_stream.dataset`
is normally namespaced or suffixed, not an exact copy of
`event.dataset`. Take it as what *this* Elastic Agent version and
input type did in *this* build — not a documented ECS guarantee. That
gap is exactly why you check your own `rubydebug` output instead of
trusting either this excerpt or the plugin's docs. If yours shows the
value only under `data_stream.dataset`, with `event.dataset` empty or
absent, change the filter's condition to
`[data_stream][dataset] == "windows.forwarded"` instead — the `stdout`
output above is how you catch that before it ships, not after.)

**This exact filter went through three real, live findings before
landing where it is now** — not a hypothetical example, an actual
history:

1. **The tier-tagging logic above originally matched a bare
   `/LogDrop/` regex.** It catches both IIS and DNS paths — they both
   live under `\\WEF01\LogDrop\<host>\...` — and mislabeled every DNS
   event as `"iis"`. This ran unnoticed in production for a full day.
   **Lesson**: query your tagged data occasionally. Don't assume a
   filter you wrote once still does what you think.
2. **The first attempt to fix it made things worse.** A path-segment
   regex like `/LogDrop\\[^\\]+\\IIS\\/` — a literal backslash right
   up against the closing `/` delimiter — broke Logstash's own config
   parser (`LogStash::ConfigurationError`, "Expected one of [...]
   after filter {"). A pipeline that fails to compile exits entirely,
   so this took the *whole pipeline* down, not just the tier-tagging
   feature. A bare substring match (`/IIS/`, `/DNS/`) sidesteps the
   escaping problem and is simpler besides — when a regex only needs
   to find a substring, don't reach for a more "precise" pattern that
   adds an escaping trap for no real benefit. `--config.test_and_exit`
   (below) would have caught this in seconds instead of a live
   crash-and-restart cycle. That bare-substring version isn't what
   ships, though — it traded the escaping issue for a different
   collision problem, fixed in finding 3 below. Don't stop reading
   here and adopt it as-is.
3. **The bare substring fix still had a real tradeoff, caught in
   review rather than in production.** On a **host-first** layout
   (`\\WEF01\LogDrop\<host>\IIS\...`), a host literally named
   `IIS-SERVER-01` would have its *DNS* logs land under
   `\\WEF01\LogDrop\IIS-SERVER-01\DNS\...`. Since the `IIS` branch is
   checked first, that path gets mislabeled `iis` anyway — a bare
   `/IIS/` substring doesn't care which path segment it actually hit.
   Hoping nobody ever names a host that way isn't a real fix for a
   pattern other people will build from.

   **The actual fix removes the ambiguity from the path layout itself
   — it doesn't patch around it in the regex.** Put the type *before*
   the hostname instead of after: `\\WEF01\LogDrop\IIS\<host>\...` and
   `\\WEF01\LogDrop\DNS\<host>\...` (what the Phase 5 drop-share layout
   uses now). With type first, `LogDrop\IIS` and `LogDrop\DNS` can only
   ever be that literal directory boundary — no hostname, however it's
   spelled, can land between `LogDrop` and the type segment right
   after it. The filter matches that anchored substring
   (`/LogDrop\\IIS/`, `/LogDrop\\DNS/`) instead of a bare `/IIS/` —
   still a single backslash, still nowhere near the closing delimiter
   that broke the parser earlier, but genuinely collision-proof now
   regardless of what any host is named. Migrated live across all 9
   IIS hosts and both DCs — a real, guardrail-gated change on
   DC01/DC02, confirmed via `--config.test_and_exit`, the real `main`
   pipeline logging `Pipeline started`/`Pipelines running` with zero
   errors, and fresh events landing in the new
   `\\WEF01\LogDrop\IIS\<host>` / `\\WEF01\LogDrop\DNS\<host>` paths
   with correct tier tags afterward. Files already shipped under the
   old host-first paths stayed put rather than getting moved — they'd
   already been ingested, and moving them risked data loss for zero
   benefit.

**One environment-specific gotcha worth flagging generally**: extracting
either the Elastic Agent or Logstash zip on a Windows host with
Defender's real-time scanning on will crawl. Defender scanning every
one of the thousands of small files inside a JVM-bundled package is a
well-known performance killer. An extraction exclusion for the install
path, plus .NET's
`[System.IO.Compression.ZipFile]::ExtractToDirectory` instead of
`Expand-Archive`, turns a stalled multi-hour extraction into a
few-second one.

## Enrichment: ATT&CK labels from Sysmon rule names

The tier-tagging filter above is already real enrichment — tagging
each event with which collection path it arrived through. The other
concrete example worth showing is the one this whole build was set up
for back in Phase 2: turning `sysmon-modular`'s ATT&CK-labeled rule
names into structured fields, which is exactly the kind of thing
approach A's Logstash filters can do that approach B's Beats-only
processors realistically can't.

Recall the `RenderedText` caveat from Phase 2: Sysmon's `RuleName`
arrives embedded in the rendered `message` text, not as its own field,
so this needs a `grok` pass against that text rather than a plain
field reference. `sysmon-modular`'s rule names embed the ATT&CK
technique directly (e.g. `technique_id=T1055,technique_name=Process
Injection`), so a second grok stage against the extracted rule name
gets you there:

```ruby
filter {
  if [winlog][channel] == "Microsoft-Windows-Sysmon/Operational" {
    grok {
      # \r?\n, not a bare \n - RenderedText line endings are CRLF on
      # Windows. A bare \n still "matches" against CRLF text since
      # grok's \n only needs the LF half, but it leaves a trailing \r
      # stuck on the end of sysmon_rule, silently polluting whatever
      # that field flows into downstream.
      match => { "message" => "RuleName:\s*%{DATA:sysmon_rule}\r?\n" }
    }
    if [sysmon_rule] and [sysmon_rule] != "-" {
      grok {
        match => {
          "sysmon_rule" => "technique_id=%{DATA:[attack][technique]},technique_name=%{DATA:[attack][technique_name]}"
        }
        tag_on_failure => []   # rules with no ATT&CK mapping shouldn't error, just skip enrichment
      }
    }
  }
}
```

`tag_on_failure => []` matters here — plenty of legitimate Sysmon rules
(especially generic ones like `RuleName: -`) won't match the
`technique_id=...` shape at all, and the default `_grokparsefailure`
tag on every one of those would drown out genuinely useful tags with
noise. Verify this the same way as the tier-tagging filter: a
`stdout { codec => rubydebug }` on a real Sysmon event, checking for
`attack.technique` in the output, before trusting it against the full
volume.

The cost of that quiet failure mode: once `tag_on_failure => []` is in
place, a future `sysmon-modular` release that changes its rule-name
format wouldn't raise any error at all — every event would just quietly
stop getting `attack.technique` populated. Nothing here catches that on
its own; it takes noticing the ATT&CK fields went empty, the same way
the tier-mislabeling issue above (the tier-tagging filter's own "three
real, live findings") went unnoticed for a full day. Query your
enriched fields occasionally, not just when something's visibly
broken.

## Phase 7 — Verify approach A actually works

Trigger one identifiable event per source tier and confirm it lands the
whole way through: source's local Event Log → WEF01's own
`ForwardedEvents` channel → Elastic Agent → Logstash's output file. Do
this independently for the DC tier, syslog, IIS, and DNS — each
mechanism can silently fail on its own without affecting the others.

The first hop, confirmed: WEF01's own local `ForwardedEvents` channel,
7,119 events in and climbing, a Sysmon registry-delete event forwarded
from MECM02 —

![WEF01's local ForwardedEvents channel with a real Sysmon event forwarded from MECM02]({{ '/assets/img/gallery/wef01-forwarded-events-sysmon.png' | relative_url }})
_Event 12, `RuleName: -` — one of the generic rules Enrichment's `tag_on_failure` is there to skip quietly instead of tagging as a parse failure_

**A verification issue worth naming**: searching Logstash's output for
`"hostname":"WKS01"` on a winlog-sourced event will find nothing, even
when the pipeline is completely healthy — that field always shows the
agent host (WEF01), not the original source. The field that actually
carries the source computer is `winlog.computer_name`. If your first
verification query comes back empty, check you're searching the right
field before concluding the pipeline is broken.

**A second one, worth naming just as clearly**: `Get-ChildItem`'s
reported file size and last-write-time on an actively-written output
file can lag well behind what's actually on disk, if the process
holding it open (Logstash's JVM, a beat, anything) buffers writes.
A file that looks frozen at some old size for hours might be
completely healthy — read its actual tail with `Get-Content -Tail`
before concluding a pipeline has stalled. Trusting `Get-ChildItem`
alone here produced a full false alarm during this build: an apparent
multi-hour "pipeline stopped" turned out to be nothing more than stale
directory metadata on a file Logstash was actively writing to every
second.

---

## Shipping layer, approach B: Winlogbeat + Filebeat

Everything above builds one complete, working pipeline. This section
builds a second one, side by side with the first, using the more
traditional single-purpose Beats instead of Elastic Agent — the
approach you're more likely to be handed a doc for if Elastic Agent
isn't your organization's standard, or if you just want something
lighter-weight and don't need Fleet-style centralized management at
all.

**Why you might pick this over Elastic Agent**: each Beat is smaller,
simpler, and does exactly one job — Winlogbeat only ever reads Windows
Event Logs, Filebeat only ever tails files/network inputs. There's less
abstraction between you and what's actually happening, which makes it
a good choice for anyone building their first log-shipping pipeline
and wanting a lower conceptual jump. The tradeoff is you're running two
separate processes with two separate config files where Elastic Agent
gave you one — and you lose Elastic Agent's ability to unify config
management under Fleet if you later want that.

Both approaches land on the same collector (WEF01), read the same
sources, and both stop at a local file with a Kafka-shaped door left
open — the architecture decision is about the shipping layer only, not
about what gets collected.

### Step 1 — Download matching versions

Match whatever Elastic stack version you're already running elsewhere
in the environment — Elastic ships Elasticsearch, Kibana, Elastic
Agent, Logstash, and every Beat on the same version train (they're all
`9.5.3` in this build, matching the Elastic Agent + Logstash pair from
Phase 6), so "the stack version" and "the Beats version" are the same
number. Mixing major versions across your Elastic tooling is asking
for subtle incompatibilities later.

```powershell
$version = '9.5.3'
Invoke-WebRequest "https://artifacts.elastic.co/downloads/beats/winlogbeat/winlogbeat-$version-windows-x86_64.zip" -OutFile "C:\LabOps\tools\winlogbeat-$version-windows-x86_64.zip"
Invoke-WebRequest "https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-$version-windows-x86_64.zip" -OutFile "C:\LabOps\tools\filebeat-$version-windows-x86_64.zip"
```

### Step 2 — Extract and install as services

Same Defender-exclusion lesson from Phase 6 applies here — exclude the
extraction path first if real-time scanning is on:

```powershell
Add-MpPreference -ExclusionPath 'C:\Install', 'C:\Program Files\Winlogbeat', 'C:\Program Files\Filebeat'
Add-Type -AssemblyName System.IO.Compression.FileSystem

[System.IO.Compression.ZipFile]::ExtractToDirectory(
    'C:\Install\winlogbeat-9.5.3-windows-x86_64.zip', 'C:\Install\extract')
# If 'C:\Program Files\Winlogbeat' already exists - a re-run after a
# failed attempt - Move-Item nests the source inside it instead of
# replacing it. Clear the way first rather than discovering that later.
if (Test-Path 'C:\Program Files\Winlogbeat') {
    Remove-Item 'C:\Program Files\Winlogbeat' -Recurse -Force
}
Move-Item 'C:\Install\extract\winlogbeat-9.5.3-windows-x86_64' 'C:\Program Files\Winlogbeat'

# repeat for filebeat, then from inside each install directory:
Push-Location 'C:\Program Files\Winlogbeat'; .\install-service-winlogbeat.ps1; Pop-Location
Push-Location 'C:\Program Files\Filebeat'; .\install-service-filebeat.ps1; Pop-Location
```

Both `install-service-*.ps1` scripts register a proper Windows service
(no NSSM or third-party wrapper needed) — you get `Set-Service` /
`Start-Service` / `Get-Service` against `winlogbeat` and `filebeat`
like any other service from here on.

### Step 3 — Winlogbeat config: the Windows-events tier

Winlogbeat's whole job here is to read the same local `ForwardedEvents`
channel Elastic Agent's `winlog` input reads in Phase 6 — the two can
point at the same channel with zero conflict, since Windows Event Log
readers don't compete for or consume events.

```yaml
# C:\Program Files\Winlogbeat\winlogbeat.yml
winlogbeat.event_logs:
  - name: ForwardedEvents
    tags: [forwarded, wef]

# For now: a local rotating file, so events are visible without
# standing up Elasticsearch/Kibana - the direct analogue of Logstash's
# file output in the other pipeline.
output.file:
  path: "D:\\WinlogbeatOut"
  filename: "winlogbeat-events"
  rotate_every_kb: 102400
  number_of_files: 10

# --- Kafka placeholder --------------------------------------------
# Swap the block above for this once Kafka exists - same one-stanza
# swap point as Logstash's output in the other pipeline.
#
# output.kafka:
#   hosts: ["kafka01.myhomelab.hv.lab:9093", "kafka02.myhomelab.hv.lab:9093", "kafka03.myhomelab.hv.lab:9093"]
#   topic: "wef01-winlogbeat-events"
#   partition.round_robin:
#     reachable_only: false

logging.level: info
logging.to_files: true
logging.files:
  path: "D:\\WinlogbeatOut\\logs"
  name: winlogbeat
  keepfiles: 7
```

Start it and confirm real content is landing — don't trust the service
status alone (see the `Get-ChildItem` lag warning from Phase 7; it
applies here identically):

```powershell
Restart-Service winlogbeat -Force
Get-Content 'D:\WinlogbeatOut\winlogbeat-events-<today>.ndjson' -Tail 5
```

A genuinely healthy Winlogbeat produces lines like this within seconds
of restart — this one is a Sysmon registry event that rode the
member-server subscription from Phase 3, exactly the way it would
through the Elastic Agent pipeline:

```json
{"@timestamp":"2026-09-12T07:59:15.505Z","winlog":{"channel":"Microsoft-Windows-Sysmon/Operational","event_id":"12", ...},"tags":["forwarded","wef"], ...}
```

Sysmon isn't the only thing riding this channel, and it's worth
confirming a non-Sysmon event looks right too — here's a real Security
4624 (successful logon) from `SQL02`, forwarded the same way, with the
verbose `message` field trimmed for space:

```json
{"@timestamp":"2026-09-12T19:51:58.501Z","winlog":{"channel":"Security","event_id":"4624","provider_name":"Microsoft-Windows-Security-Auditing","computer_name":"SQL02.myhomelab.hv.lab","event_data":{"TargetUserName":"MECM01$","TargetDomainName":"MYHOMELAB","LogonType":"3","IpAddress":"-"}},"event":{"outcome":"success","action":"Logon","code":"4624"},"host":{"name":"SQL02.myhomelab.hv.lab"},"tags":["forwarded","wef"]}
```

Same shape either way: `winlog.channel` and `winlog.event_id` tell you
what happened, `host.name` (or `winlog.computer_name` — see the Phase
7 field-name warning above) tells you where, and `tags` confirms it
rode the forwarding pipeline rather than something read directly off
WEF01's own local logs.

### Step 4 — Filebeat config: syslog, IIS, and DNS

Filebeat covers the three non-Windows-Event sources — one `udp` input
for syslog, two `filestream` inputs for the IIS/DNS drop-share files
already being populated by Phase 5's tailer.

```yaml
# C:\Program Files\Filebeat\filebeat.yml
filebeat.inputs:
  - type: udp
    id: syslog-udp-demo
    host: "0.0.0.0:5514"
    tags: [syslog, demo]
    processors:
      - syslog:
          field: message
          format: auto
      - add_fields:
          target: wef
          fields:
            tier: syslog

  - type: filestream
    id: iis-logdrop-demo
    paths:
      - 'D:\LogDrop\IIS\*\*\*.log'
    tags: [iis, demo]
    processors:
      - add_fields:
          target: wef
          fields:
            tier: iis

  - type: filestream
    id: dns-logdrop-demo
    paths:
      - 'D:\LogDrop\DNS\*\*\*.log'
    tags: [dns, demo]
    processors:
      - add_fields:
          target: wef
          fields:
            tier: dns

output.file:
  path: "D:\\FilebeatOut"
  filename: "filebeat-events"
  rotate_every_kb: 102400
  number_of_files: 10

# --- Kafka placeholder --------------------------------------------
# output.kafka:
#   hosts: ["kafka01.myhomelab.hv.lab:9093", "kafka02.myhomelab.hv.lab:9093", "kafka03.myhomelab.hv.lab:9093"]
#   topic: "wef01-filebeat-events"
#   partition.round_robin:
#     reachable_only: false

logging.level: info
logging.to_files: true
logging.files:
  path: "D:\\FilebeatOut\\logs"
  name: filebeat
  keepfiles: 7
```

The `add_fields` on each input sets `wef.tier`, matching the field
name approach A's Logstash filter produces — without it, this
pipeline's events would carry `tags: [iis]` / `[dns]` / `[syslog]`
instead, a different output shape than approach A for no reason other
than nobody having wired it up. This is the extent of what a Beat's
processor set can do here, and it's genuinely enough for it: a static
per-input tag is a different problem than the multi-stage grok
Enrichment above needs for ATT&CK labels, which Beats' own processors
aren't built for — see that section for why.

Two things worth calling out explicitly, because both are the kind of
detail that only shows up once you actually run the thing:

**Filebeat has a built-in `syslog` input type — don't use it.** It's
deprecated as of the version this lab tested; starting it logs exactly
this warning:

```
"DEPRECATED: Syslog input. Use Syslog processor instead."
```

The current recommended shape is a plain `udp` (or `tcp`) input with a
`syslog` processor attached, which is what the config above uses — and
it's the same pattern Elastic Agent's own `syslog-udp` input already
uses in Phase 6, so learning it once covers both pipelines.

**Two Beats/agents can tail the same file with zero conflict.**
Filebeat's IIS/DNS `filestream` inputs above point at the exact same
drop-share files Elastic Agent already tails in approach A. Each
shipper keeps its own independent read-offset registry, so there's no
locking issue, no duplicate-detection problem, and no need to choose
between running one pipeline or the other while you're evaluating both
— this is exactly how this lab ran both pipelines simultaneously
against the same live drop-share without any special handling.

**If you're pointing this at a live UDP port a production listener
already owns** (this lab's real syslog senders were already wired to
Elastic Agent's listener on UDP/514), don't fight over the port — bind
Filebeat's demo listener to a different one (5514 here) and open the
matching inbound firewall rule:

```powershell
New-NetFirewallRule -DisplayName 'Filebeat Demo Syslog UDP 5514' `
    -Direction Inbound -Protocol UDP -LocalPort 5514 -Action Allow
```

To make this pipeline authoritative instead of a side-by-side demo,
repoint your actual syslog senders at the new port and retire the other
pipeline's listener — nothing else about the config needs to change.

### Step 5 — Start both services and verify

```powershell
Set-Service winlogbeat -StartupType Automatic
Set-Service filebeat -StartupType Automatic
Restart-Service winlogbeat -Force
Restart-Service filebeat -Force
Get-Service winlogbeat, filebeat
```

Don't stop at "Running" — that's exactly the lesson this whole build
keeps teaching in different forms. Filebeat's own metrics log (emitted
every 30 seconds to its log file) is the fastest way to confirm each
input is actually processing events, without needing to dig through
potentially huge output files:

```powershell
Get-Content 'D:\FilebeatOut\logs\filebeat-<today>.ndjson' -Tail 50 |
    Select-String 'events_pipeline_published_total','processor'
```

A healthy syslog input, right after a test packet, looks like this
in that snapshot — one event received, matched against the syslog
processor, and published:

```json
"syslog-udp-demo":{"device":"0.0.0.0:5514","events_pipeline_published_total":1,
  "received_bytes_total":84,"received_events_total":1, ...}
"processor":{"syslog":{"1":{"success":1}}}
```

Send yourself a real test packet to confirm end to end, rather than
waiting for real traffic:

```powershell
$msg = '<134>Sep 12 08:10:00 test-sender demo: verification packet'
$udp = New-Object System.Net.Sockets.UdpClient
$bytes = [System.Text.Encoding]::ASCII.GetBytes($msg)
$udp.Send($bytes, $bytes.Length, '127.0.0.1', 5514) | Out-Null
$udp.Close()
```

### A standalone listener, for when you need to rule out the shipper entirely

The test packet above proves Filebeat received something, but if it
*doesn't* show up, you're left guessing whether the problem is the
network path, a firewall rule, or Filebeat's own config. A tiny
standalone UDP listener — no Beats, no Elastic Agent, nothing but raw
.NET sockets — answers "is anything even arriving on this port at all"
on its own, which cuts that guesswork in half:

```powershell
# Stop Filebeat first (or use a different port) - only one process can
# bind a given UDP port for receiving at a time.
$listener = New-Object System.Net.Sockets.UdpClient(5514)
$remoteEndpoint = New-Object System.Net.IPEndPoint([System.Net.IPAddress]::Any, 0)

Write-Host "Listening on UDP/5514 - Ctrl+C to stop"
try {
    while ($true) {
        $bytes = $listener.Receive([ref]$remoteEndpoint)
        $text  = [System.Text.Encoding]::ASCII.GetString($bytes)
        Write-Host "[$(Get-Date -Format 'HH:mm:ss')] from $($remoteEndpoint.Address): $text"
    }
}
finally {
    $listener.Close()
}
```

Run this on WEF01 in place of Filebeat, then fire the same test-packet
snippet from above (or point a real syslog sender at it) from another
machine. If the message shows up here, the network path and firewall
rule are both fine and any remaining problem is in Filebeat's own
input config — a smaller, more specific thing to troubleshoot than "syslog
isn't working." If it doesn't show up, you've just ruled Filebeat out
entirely and can go straight to checking routing and firewall rules
instead of staring at a YAML file that was never the problem.

## Comparing the two, now that you've built both

### Elastic Agent + Logstash

- **Processes to manage**: 2 (Agent, Logstash)
- **Config surface**: one YAML per input type, plus one Logstash
  pipeline file
- **Centralized management later (Fleet)**: built in, if you ever
  stand up Fleet/Kibana
- **Resource footprint**: Logstash's JVM is the heaviest single
  piece in this build, with or without enrichment filters running
- **Enrichment/routing before Kafka**: Logstash filters (dissect,
  aggregate, ECS mapping) — genuinely powerful
- **Kafka cutover later**: swap Logstash's one output stanza
- **Good first pipeline to learn on**: if you already know you'll
  want Logstash-side enrichment eventually — build that muscle
  memory first

### Winlogbeat + Filebeat

- **Processes to manage**: 2 (Winlogbeat, Filebeat)
- **Config surface**: one YAML per Beat
- **Centralized management later (Fleet)**: not available — no
  shared control plane
- **Resource footprint**: slightly lighter without Logstash's JVM,
  if you skip enrichment
- **Enrichment/routing before Kafka**: each Beat's own lighter
  processor set — less flexible, usually enough for straightforward
  shipping; see Enrichment above for a concrete example (the ATT&CK
  dissect) of the kind of multi-stage text parsing a Beat's
  processor set isn't really built for
- **Kafka cutover later**: swap each Beat's one output stanza
- **Good first pipeline to learn on**: if you want the simplest
  possible mental model — one Beat, one job

Both are legitimate answers to "how do I ship these logs somewhere."
If you want it as a decision rather than a table to weigh yourself:

- **You're the only person who'll ever touch this, and you want to
  understand every byte** → Winlogbeat + Filebeat.
- **You have a team, you'll add more sources over time, and you want
  central config management** → Elastic Agent + Logstash.
- **You already run Fleet somewhere else in your environment** →
  Elastic Agent, so this pipeline can join the same fleet later instead
  of being a permanent outlier.
- **You're air-gapped, resource-constrained, or can't justify running a
  JVM for this** → Winlogbeat + Filebeat.

## Common failure modes, quick reference

Everything below was a real symptom hit during this build. If you're
troubleshooting either pipeline, check here before diving deep — the
fix is usually smaller than the symptom suggests.

**`Get-WinEvent -ListLog` on a Sysmon channel returns "log not found"**
- Cause: wrong channel name — `Microsoft-Windows-Sysinternals-Sysmon/Operational`
  instead of the real `Microsoft-Windows-Sysmon/Operational`
- Fix: `wevtutil el | Select-String Sysmon` to get the real name

**Subscription forwards everything except Sysmon; WSMan fault 1818**
- Cause: Sysmon's channel ACL has no `NETWORK SERVICE` grant
- Fix: ACL fix in Phase 2

**New group member gets "Access is denied" (WSMan fault 5) on a subscription that already works for others**
- Cause: Kerberos ticket predates the group membership change
- Fix: reboot the machine, then `wecutil rs`

**Every non-admin source gets "Access is denied" reaching the collector at all**
- Cause: Event Forwarding Plugin's own WinRM SDDL has no `Authenticated Users` ACE
- Fix: SDDL fix in Phase 3

**DNS Analytical channel can't be queried or subscribed to while enabled**
- Cause: direct-channel type; no external consumer can touch it live
- Fix: debug text-file logging instead (Phase 5)

**Logstash/Beats output file looks frozen at some old size for hours**
- Cause: `Get-ChildItem` metadata lags an actively-written file
- Fix: `Get-Content -Tail` the actual file instead

**Searching for a source hostname in Logstash's output finds nothing**
- Cause: wrong field — `hostname` is always the agent host, not the source
- Fix: search `winlog.computer_name` instead

**Zip extraction for Elastic Agent/Logstash crawls for hours**
- Cause: Defender real-time scanning every extracted file
- Fix: exclude the path first, use `ZipFile.ExtractToDirectory`

**Filebeat logs `"DEPRECATED: Syslog input"` on startup**
- Cause: using the built-in `syslog` input type
- Fix: switch to `udp`/`tcp` input + `syslog` processor

**Two shippers pointed at the same UDP port both fail to bind**
- Cause: only one process can own a UDP port for receiving at a time
- Fix: pick different ports, or stop one before testing the other

**A DNS event is tagged `iis` (or vice versa) despite substring matching on the tier segment**
- Cause: host-first drop-share layout (`LogDrop\<host>\IIS`) lets a
  host's own name collide with the other type's substring match
- Fix: type-first layout (`LogDrop\IIS\<host>`) instead — see Phase 5/6

**Two tailer instances on the same host silently lose byte-offset progress**
- Cause: both defaulted to the same hardcoded state file (only happens
  on a host running more than one tailer instance, like DC01 tailing
  both IIS and DNS)
- Fix: distinct `-StateFilePath` per instance

**Logstash stops processing events, but `Get-Service`/`Get-Process` both show it running**
- Cause: JVM heap exhausted (`OutOfMemoryError: Java heap space`) —
  pipeline worker threads are dead, but the process itself is still alive
- Fix: set `-Xms`/`-Xmx` in `config/jvm.options`, confirm via the live
  process's command line — see Sizing and retention

## What's next

- Kafka doesn't exist in this lab yet — both pipelines above are built
  with that swap in mind, but until it's real, "local rotating file" is
  the actual, load-bearing output for both.
- WEF01 now has the MECM client and SCOM agent installed, the same way
  every other server in this lab does — a log collector that nobody's
  watching is just a second thing that can silently fail. Next is
  adding it to the SUPERLAB monitoring dashboard with checks specific
  to what this box actually does: all four shipper services (Elastic
  Agent, Logstash, Winlogbeat, Filebeat) actually running rather than
  just installed, and subscription health — events-per-second per WEC
  subscription, so a tier going quiet shows up as a graph dropping to
  zero instead of a discovery made by "check on it" days later.
- If you build the Winlogbeat/Filebeat pipeline as your only pipeline
  rather than side by side with Elastic Agent, you can drop the demo
  port (5514) and repoint your real syslog senders directly at
  Filebeat instead.

## If something doesn't match this guide

Every claim in this post came from checking the actual system state,
not from assuming a service's "Running" status meant data was flowing
— that distinction cost real troubleshooting time more than once
during this build: on the Sysmon channel, on the Logstash heap crash
that `Get-Process` never let on about, and later on what turned out to
be nothing more than a stale file-size reading on a perfectly healthy
Logstash output. If a step here doesn't produce what it says it should
in your environment, read the actual content the pipeline is producing
before trusting either the service status or a directory listing.
