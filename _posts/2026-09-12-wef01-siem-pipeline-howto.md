---
title: "Homelab Build-Out — Centralized Log Collection with WEF01, Two Ways"
author: uzrg
date: 2026-09-12 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Microsoft, HyperV, Windows Server]
tags: [Windows Event Forwarding, Elastic, Winlogbeat, Filebeat, Logstash, Sysmon, Syslog, IIS, DNS, How-To, Tutorial]
pin: false
mermaid: false
---

# A step-by-step guide: Windows Event Forwarding + a SIEM-shaped pipeline, built two different ways

Every server in SUPERLAB up to this point has kept its own Event Log,
its own IIS log files, its own DNS debug output — useful if you already
know which machine to look at, useless if you don't. This guide builds
`WEF01`, a single collector that every domain-joined Windows machine
forwards its events to natively, extended to also pull in Linux syslog
and file-based logs (IIS, DNS) that Windows Event Forwarding itself
can't reach.

Then it builds the same pipeline a second time, with different tools,
specifically so you can compare them side by side. That second build is
the real point of this post: if you're a junior engineer handed either
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
- IIS and DNS logs — both file-based, neither reachable by native Event
  Forwarding — shipped through a near-real-time file-tailing pattern
- **Two parallel, independently working shipping pipelines** on WEF01
  itself: one built on Elastic Agent + Logstash, one built on
  Winlogbeat + Filebeat, both writing to local rotating files with a
  clearly marked spot to swap in Kafka once that exists
- A pipeline you've verified actually moves data, not just one where
  every service says "Running"

## Architecture

```
  Domain Controllers OU  --[GPO: WEF-Forwarding-DCs]-->              \
  Servers OU             --[GPO: WEF-Forwarding-MemberServers]-->     WEF01
  Workstations OU        --[GPO: WEF-Forwarding-Workstations]-->     /(ForwardedEvents)

  UBUNTU01 / future Linux, pfSense / network devices
      --[rsyslog / syslog]-->  WEF01

  IIS-hosting servers, DC01 / DC02 DNS debug logs
      --[FileSystemWatcher tail]-->  \\WEF01\LogDrop\<host>\...

                              |
                              v
             WEF01's shipping layer, built two independent ways:

               Pipeline A:  Elastic Agent -> Logstash -> file output
               Pipeline B:  Winlogbeat ----------------> file output
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
  generalized-template post if you need one)
- An OU structure to place computer/group objects into (this lab uses
  `OU=Servers,OU=LabOU` — adjust to whatever yours is)
- A Linux host you can experiment with `rsyslog` on, if you want to
  follow the syslog section
- About a full day if you're building both pipelines end to end, given
  the number of gotchas documented along the way; a few hours for just
  one

---

## Phase 1 — Provision the collector

Nothing special here beyond your normal VM build: clone from template,
join the domain, land the computer object in your servers OU (not the
default `CN=Computers` container — new AD objects should always go
somewhere deliberate). Give it a second data disk (`D:`) — you'll want
somewhere other than the OS disk for log output, the IIS/DNS drop
share, and eventually Logstash's own install.

Sizing: 2 vCPU / 4 GB is enough to start. The JVM inside Logstash is
the single biggest memory cost in either pipeline; watch it if you add
volume later.

## Phase 2 — Sysmon, fleet-wide, with ATT&CK labels

Deploy Sysmon before building the subscriptions in Phase 3, so the
subscription queries can include its channel from day one instead of
being edited again later.

- **Config choice**: [Olaf Hartong's `sysmon-modular`](https://github.com/olafhartong/sysmon-modular)
  over the more common SwiftOnSecurity baseline, specifically because
  it embeds MITRE ATT&CK Tactic/Technique IDs directly into each rule's
  name — a simple field dissect downstream gets you structured
  `attack.tactic` / `attack.technique` fields for free. This lab used
  the repo's `sysmonconfig-with-filedelete.xml` variant for the added
  file-delete visibility. Pin the exact commit you pull the config
  from, same as you'd pin the Sysmon binary version.
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

Security's SDDL includes `(A;;CC;;;NS)`; Sysmon's does not include any
`;NS)` ACE at all. Fix it (idempotent, safe to re-run on a channel that
already has the grant):

```powershell
$channel = 'Microsoft-Windows-Sysmon/Operational'
$sddl = (wevtutil gl $channel | Select-String 'channelAccess').ToString() -replace 'channelAccess:\s*', ''
if ($sddl -notmatch ';NS\)') {
    wevtutil sl $channel "/ca:$($sddl)(A;;0x1;;;NS)"
}
```

The general lesson under both of these: when something reads fine
locally as Administrator/SYSTEM but a service or subscription can't
consume it, look at what identity that service actually runs as, and
diff its permissions against something that already works. That
technique found this in minutes once applied; guessing at "maybe the
config is wrong" or "maybe reinstall it" cost a lot more time first.

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
  `Server=http://WEF01.yourdomain.tld:5985/wsman/SubscriptionManager/WEC,Refresh=900`
- Computer Config → Admin Templates → Windows Components → Windows
  Remote Management (WinRM) → WinRM Service → **Allow remote server
  management through WinRM**: Enabled

And a matching subscription created via `wecutil cs` with an XML
definition scoping `AllowedSourceDomainComputers` to that tier's AD
group (or the built-in `Domain Controllers` group for the DC tier).
Here's the real member-server subscription from this build, with the
group SID generalized — the DC and workstation subscriptions are the
same shape, just a different `Query` and a different group:

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
  <AllowedSourceDomainComputers>O:NSG:NSD:(A;;GA;;;<group-SID>)</AllowedSourceDomainComputers>
</Subscription>
```

Resolve `<group-SID>` from the AD group itself rather than typing one
in by hand — it changes if the group is ever recreated:

```powershell
$sid = (Get-ADGroup 'WEF-MemberServers').SID.Value
```

Save the filled-in XML as `WEF-MemberServers.xml` and create the
subscription from it:

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
`4722`/`4724`/`4738` account changes, `4662` directory object access,
plus the whole `Directory Service` log and, once enabled, DNS's
Analytical channel) alongside the same base logon events; the
workstation tier stays deliberately narrow — logon events plus
Sysmon — since endpoint telemetry is what matters there, not
account-management noise a workstation rarely generates in the first
place.

### A subtle failure that looks like a permissions bug but isn't

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

**Fix**: reboot the machine after adding it to the forwarding group,
then force a retry:

```powershell
gpupdate /force /target:computer
Restart-Service EventLog
wecutil rs <SubscriptionName>
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
DACL always ends in this SDDL shape.

## Phase 4 — Linux and network-device syslog

No agent on the Linux side at all — the same "nothing extra runs on the
source" principle as everything else in this build. Point `rsyslog` at
WEF01:

```
# /etc/rsyslog.d/90-wef01.conf on the Linux host
*.* @@wef01.yourdomain.tld:514   # TCP; use a single @ for UDP
```

Network devices that support remote syslog natively (most firewalls,
switches) just need their syslog destination pointed at WEF01 the same
way — if your firewall is a shared, hands-off appliance in your
environment the way pfSense is in this lab, that's a change for
whoever owns it, not something to make unilaterally.

## Phase 5 — File-based logs: IIS and DNS, near-real time

Windows Event Forwarding can't reach IIS log files or DNS's debug text
log — they're not Event Log channels. Rather than polling for rotated
files every few minutes (real detection latency cost), this uses a
`FileSystemWatcher`-based tailer that reacts to writes as they happen:

- IIS opens its log files with share-read access, so the currently
  active file can be tailed while IIS is still writing to it.
- The tailer tracks a per-file byte offset in a small local state file
  (survives restarts) and appends only new bytes to
  `\\WEF01\LogDrop\<hostname>\IIS\`.
- Run it as a Scheduled Task, "At startup," restart-on-failure.
- DNS's live Analytical channel turns out to be a dead end for
  forwarding — see the callout below — so DNS uses the classic debug
  text-file log (`Set-DnsServerDiagnostics -EnableLoggingToFile`)
  shipped through the exact same tailer/drop-share pattern as IIS.

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
    [string]$DropShare
)

$LogDirectories = $LogDirectoriesRaw -split ';'
$stateFile = 'C:\LabOps\tailer-state.json'
$sweepIntervalSeconds = 15

# Shared across the main thread and event-handler threads, so both the
# Changed-event handler and the periodic sweep read/write the same
# offsets without racing each other.
$sync = [hashtable]::Synchronized(@{})
if (Test-Path $stateFile) {
    (Get-Content -Raw $stateFile | ConvertFrom-Json).PSObject.Properties |
        ForEach-Object { $sync[$_.Name] = [int64]$_.Value }
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
        ($Sync | ConvertTo-Json) | Set-Content -Path $StateFile -Encoding utf8
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
    # used here - a real bug hit during this build, caught only
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
    '-NoProfile -ExecutionPolicy Bypass -File C:\LabOps\Start-LogTailer.ps1 -LogDirectoriesRaw "C:\inetpub\logs\LogFiles\W3SVC1" -DropShare "\\WEF01\LogDrop\DEVOPS01\IIS"'
$trigger = New-ScheduledTaskTrigger -AtStartup
$settings = New-ScheduledTaskSettingsSet -RestartCount 999 -RestartInterval (New-TimeSpan -Minutes 1)
Register-ScheduledTask -TaskName 'LogTailer' -Action $action -Trigger $trigger `
    -Settings $settings -User 'SYSTEM' -RunLevel Highest
```

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

Phase 1 suggested 2 vCPU / 4 GB as a starting point. In practice this
lab's WEF01 is provisioned with roughly 3.2 GB total — under the 4 GB
suggested above, not "3.2 GB used out of 4 GB." That gap mattered: it's
part of why Logstash's JVM ran out of heap for real during this build
(see the heap section below) once WEF01 picked up its own MECM/SCOM
monitoring agents on top of everything else running here. If you're
only running one pipeline (the realistic production choice, not this
post's side-by-side demo), match the VM's actual provisioned memory to
4 GB as originally suggested rather than assuming it landed there;
budget 6–8 GB if you genuinely want both pipelines running at once the
way this build does, especially once the box is itself a monitored
endpoint generating its own telemetry.

**Disk**: the `D:` drive in this build is 60 GB, holding Logstash's
install, both pipelines' rotating output, and the IIS/DNS drop-share.
On a single, fairly busy day mid-build, Logstash's own file output
(`wef-events-<date>.log`, one file per day, no cap on that file's size)
reached **8 GB** before rolling to the next day — that number scales
directly with source volume and how many Sysmon-generating hosts you
have, so treat it as a starting estimate, not a hard ceiling. The two
Beats pipelines are bounded by contrast: `rotate_every_kb: 102400` ×
`number_of_files: 10` caps each Beat's own output at **roughly 1 GB**
of retained history, oldest file dropped as new ones roll in. If you
want Logstash's file output similarly bounded rather than growing
per day, add a size-based rotation policy to it the same way, or park
a cleanup Scheduled Task on `D:\LogstashOut\` the same way the
drop-share needs one (see below).

**Logstash's JVM heap — this one is a real incident, not a
hypothetical**: this build's `jvm.options` originally had no explicit
`-Xms`/`-Xmx`, defaulting to 1 GB. That default genuinely ran out —
`java.lang.OutOfMemoryError: Java heap space`, fatal on both pipeline
worker threads, Logstash dead — once WEF01's own Sysmon volume spiked
after the MECM/SCOM agents landed on WEF01 itself (see "What's next"):
a box that's also a monitored endpoint generates its own Sysmon
telemetry on top of everything it's collecting from the rest of the
fleet, and 1 GB wasn't enough headroom for that combined load. The
practical lesson: don't treat a heap-sizing recommendation (including
this post's own numbers) as fixed — watch actual JVM memory under real
load and raise it before you hit the wall, not after Logstash has
already gone down silently for hours (`Get-Process java` still showed
a running process throughout; only the log's `FATAL` entries revealed
anything was wrong — another entry for the "status ≠ actually working"
pile this whole build keeps adding to).

Set the heap directly in `config/jvm.options` (note: a separate
`jvm.options.d/heap.options` file did **not** get picked up in this
Logstash build — verify your version actually reads that directory
before relying on it, and check with the real running process's
command line, not just the file you wrote):

```
-Xms1536m
-Xmx1536m
```

1536 MB comfortably covers this pipeline's real volume including the
Sysmon spike that caused the original crash; raise it further if you
add enrichment filters (Phase 6's Enrichment section) or higher-volume
sources later. Confirm the value actually took effect by checking the
live process, not the config file:

```powershell
(Get-CimInstance Win32_Process -Filter "Name='java.exe'").CommandLine
```

And validate any config change before restarting the real service —
this would have caught the parser-breaking regex above in seconds
instead of costing a crash-and-restart cycle:

```powershell
& 'D:\Logstash\bin\logstash.bat' -f 'D:\Logstash\config\wef01-pipeline.conf' --config.test_and_exit
```

**DNS debug logging volume**: the workaround in Phase 5 — classic
`Set-DnsServerDiagnostics -EnableLoggingToFile` — is genuinely verbose.
Every query gets a line, not just the interesting ones, and a busy
resolver can produce multiple gigabytes per day. Treat it the same way
as any other debug-level logging you'd never leave on in a
non-troubleshooting context on a production DNS server: fine here
because DC01/DC02 aren't under real query load, worth a second thought
(sampling, shorter retention, or scoping to specific event categories
via the diagnostics cmdlet's other switches) before doing the same on
a busier resolver.

**Drop-share retention**: the tailer only ever appends — it never
deletes anything from `\\WEF01\LogDrop\`. Pair it with a separate,
simple cleanup Scheduled Task that purges files older than a
conservative window (24–48h is plenty, since Elastic Agent/Filebeat's
own read-offset tracking is what actually prevents re-ingestion, not
how long the drop-share copy survives):

```powershell
Get-ChildItem 'D:\LogDrop' -Recurse -File |
    Where-Object LastWriteTime -lt (Get-Date).AddHours(-48) |
    Remove-Item -Force
```

## Firewall rules, all in one place

Every port this build actually needs open, gathered here instead of
scattered across phases:

**TCP 5985, inbound on WEF01** — WinRM: subscription manager + event
push from every forwarding source.

**UDP/TCP 514, inbound on WEF01** — syslog from Linux/network devices
(Phase 4).

**UDP 5514, inbound on WEF01** — Filebeat's demo syslog listener
(Phase 6B only; skip it if you're not running the side-by-side
comparison).

**TCP 5044, loopback only on WEF01** — Beats → Logstash, never leaves
the host.

None of these need opening anywhere except WEF01 itself — every source
machine only makes outbound connections (to push events, forward
syslog, or write to the drop-share), so there's nothing to open on
DC01/DC02, the member servers, or the workstations. Outbound 5985 from
each source is what actually carries that traffic, and it's usually
allowed by default on a Windows host firewall — but if yours is locked
down more tightly than the out-of-the-box profile, confirm outbound
5985 explicitly rather than assuming it's open just because inbound is
covered on WEF01's side.

One more thing worth knowing if WinRM has never been touched on
WEF01 before this build: `winrm quickconfig -force` (Phase 3) is what
actually binds the WinRM listener to the network interface in the
first place — a fresh Windows install has WinRM's *service* running,
but its listener defaults to effectively loopback-only until
`quickconfig` (or the equivalent GPO-driven listener creation) creates
a real HTTP listener and opens the matching firewall rule. Running
`wecutil qc` alone, without `winrm quickconfig`, is a common way to end
up with a subscription that looks correctly configured but has no
listener for anything to actually reach.

## Production hardening (out of scope here, but worth knowing)

Everything in this build is unencrypted, which is a reasonable
trade-off in an isolated lab and not one to carry into anything
internet-facing or handling real user data:

- **WinRM** runs over plain HTTP (5985) here. Production wants HTTPS
  (5986) with a real certificate, which also means reworking the
  SubscriptionManager GPO value and the WinRM listener config to match.
- **Syslog** over UDP/TCP 514 is unencrypted and, on UDP, unauthenticated
  — anyone who can reach the port can inject events. TLS-wrapped syslog
  (RFC 5425) or an IPsec-protected segment closes that gap.
- **The Beats protocol** between Elastic Agent/Winlogbeat/Filebeat and
  Logstash is loopback-only in this build, which sidesteps the problem
  entirely — but the moment Logstash lives on a different host than its
  shippers, that link needs TLS (`ssl_enabled` on both the beats input
  and each shipper's output) rather than crossing a network in the
  clear.

None of this blocks anything in this guide — it's what to add before
this pattern leaves a lab.

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
        paths: ['D:\LogDrop\*\IIS\*\*.log']

  - type: filestream
    id: dns-logdrop
    streams:
      - data_stream: { dataset: dns.debug }
        paths: ['D:\LogDrop\*\DNS\*\*.log']
```

Elastic Agent runs as a Windows service (`Elastic Agent`) once
installed — the standalone install places its binary and this config
at `C:\Program Files\Elastic\Agent\elastic-agent.yml`. Standalone mode
needs no separate "enrollment" step the way Fleet-managed agents do:
drop the config in place and (re)start the service, and it starts
running the inputs defined in the file immediately —
`elastic-agent.exe status` should report `(HEALTHY) Running` with
`fleet (STOPPED, Not enrolled)`, which is expected and correct for this
mode, not an error to chase.

Logstash's own pipeline, for now, is deliberately boring: `beats` input
on 5044 → a `file` output with size-based rotation. That output stanza
is the one thing that changes when Kafka exists later — nothing
upstream of it needs to know or care. This build's scheduled task
launches Logstash with `-f D:\Logstash\config\wef01-pipeline.conf`
directly — a real gotcha in its own right: passing `-f` on the command
line makes Logstash **ignore `pipelines.yml` entirely** (it logs
`Ignoring the 'pipelines.yml' file because command line options are
specified`), so if you're used to multi-pipeline setups via
`pipelines.yml`, a single `-f` flag silently overrides that whole
mechanism rather than adding to it.

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
  # IIS and DNS both land under \\WEF01\LogDrop\<host>\..., so a bare
  # /LogDrop/ match catches both and mislabels every DNS event as
  # "iis" - match on the "IIS"/"DNS" substring instead (see the
  # callout below for why a path-segment regex here is a trap).
  if [event][dataset] == "windows.forwarded" {
    mutate { add_field => { "[wef][tier]" => "windows_event_forwarding" } }
  } else if [event][dataset] =~ /^syslog/ or [log][source][address] {
    mutate { add_field => { "[wef][tier]" => "syslog" } }
  } else if [log][file][path] =~ /IIS/ {
    mutate { add_field => { "[wef][tier]" => "iis" } }
  } else if [log][file][path] =~ /DNS/ {
    mutate { add_field => { "[wef][tier]" => "dns" } }
  }
}

output {
  file {
    path => "D:/LogstashOut/wef-events-%{+YYYY-MM-dd}.log"
    codec => json_lines
  }
}
```

Verify a filter like this against a real event before trusting it —
don't assume a field name matches what a plugin's docs say without
checking. A one-line `stdout { codec => rubydebug }` output alongside
(or instead of) the file output for a few seconds shows you the exact
field structure Logstash is actually working with:

```
output {
  stdout { codec => rubydebug }
}
```

**Two real bugs were found in this exact filter, one of them live and
in production**:

1. **The tier-tagging logic above originally matched a bare
   `/LogDrop/` regex**, which catches both IIS and DNS paths (they
   both live under `\\WEF01\LogDrop\<host>\...`) and mislabeled every
   DNS event as `"iis"`. This ran unnoticed in production for a full
   day before being caught — a good argument for actually querying
   your tagged data occasionally rather than assuming a filter you
   wrote once still does what you think.
2. **The first attempt to fix it made things worse.** A path-segment
   regex like `/LogDrop\\[^\\]+\\IIS\\/` — matching a literal backslash
   right up against the closing `/` delimiter — broke Logstash's own
   config parser (`LogStash::ConfigurationError`, "Expected one of
   [...] after filter {"), and since a pipeline that fails to compile
   makes Logstash exit entirely, this took the *whole pipeline* down,
   not just the tier-tagging feature. The bare substring match
   (`/IIS/`, `/DNS/`) above sidesteps the entire escaping problem and
   is simpler besides — when a regex only needs to find a substring,
   don't reach for a more "precise" pattern that adds an escaping trap
   for no real benefit. `--config.test_and_exit` (below) would have
   caught this in seconds instead of costing a live crash-and-restart
   cycle.

**One environment-specific gotcha worth flagging generally**: if you're
extracting either the Elastic Agent or Logstash zip on a Windows host
with Defender's real-time scanning on, expect extraction to crawl —
Defender scanning every one of the thousands of small files inside a
JVM-bundled package is a well-known performance killer. An extraction
exclusion for the install path, plus using .NET's
`[System.IO.Compression.ZipFile]::ExtractToDirectory` instead of
`Expand-Archive`, turned a stalled multi-hour extraction into a
few-second one.

## Phase 7 — Verify pipeline A actually works

Trigger one identifiable event per source tier and confirm it lands the
whole way through: source's local Event Log → WEF01's own
`ForwardedEvents` channel → Elastic Agent → Logstash's output file. Do
this independently for the DC tier, syslog, IIS, and DNS — each
mechanism can silently fail on its own without affecting the others.

**A verification bug worth naming**: searching Logstash's output for
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
#   hosts: ["kafka01.yourdomain.tld:9092"]
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

  - type: filestream
    id: iis-logdrop-demo
    paths:
      - 'D:\LogDrop\*\IIS\*\*.log'
    tags: [iis, demo]

  - type: filestream
    id: dns-logdrop-demo
    paths:
      - 'D:\LogDrop\*\DNS\*\*.log'
    tags: [dns, demo]

output.file:
  path: "D:\\FilebeatOut"
  filename: "filebeat-events"
  rotate_every_kb: 102400
  number_of_files: 10

# --- Kafka placeholder --------------------------------------------
# output.kafka:
#   hosts: ["kafka01.yourdomain.tld:9092"]
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
drop-share files Elastic Agent already tails in Pipeline A. Each
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
input config — a smaller, more specific thing to debug than "syslog
isn't working." If it doesn't show up, you've just ruled Filebeat out
entirely and can go straight to checking routing and firewall rules
instead of staring at a YAML file that was never the problem.

## Comparing the two, now that you've built both

### Elastic Agent + Logstash

- **Processes to manage**: 2 (Agent, Logstash)
- **Config surface**: one YAML per input type, plus one Logstash pipeline file
- **Centralized management later (Fleet)**: built in, if you ever stand up Fleet/Kibana
- **Resource footprint**: Logstash's JVM is the heaviest single piece either way
- **Enrichment/routing before Kafka**: Logstash filters (dissect, aggregate, ECS mapping) — genuinely powerful
- **Kafka cutover later**: swap Logstash's one output stanza
- **Good first pipeline to learn on**: if you already know you'll want Logstash-side enrichment

### Winlogbeat + Filebeat

- **Processes to manage**: 2 (Winlogbeat, Filebeat)
- **Config surface**: one YAML per Beat
- **Centralized management later (Fleet)**: not available — no shared control plane
- **Resource footprint**: slightly lighter without Logstash's JVM, if you skip enrichment
- **Enrichment/routing before Kafka**: each Beat's own lighter processor set — less flexible, usually enough for straightforward shipping
- **Kafka cutover later**: swap each Beat's one output stanza
- **Good first pipeline to learn on**: if you want the simplest possible mental model — one Beat, one job

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
- Cause: wrong channel name — `Sysinternals-Sysmon` instead of `Sysmon`
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
not from assuming a service's "Running" status meant data was flowing —
that distinction cost real debugging time more than once during this
build, on both the Sysmon channel and, later, on what turned out to be
nothing more than a stale file-size reading on a perfectly healthy
Logstash output. If a step here doesn't produce what it says it should
in your environment, read the actual content the pipeline is producing
before trusting either the service status or a directory listing.
