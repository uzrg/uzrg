---
title: "Homelab Build-Out — Standing Up SCOM 2025, Then Making It Highly Available"
author: uzrg
date: 2026-08-13 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Microsoft, HyperV, Windows Server]
tags: [Microsoft, SCOM, System Center, Monitoring, MECM, Active Directory, SQL Server, PKI, High Availability, HAProxy, pfSense, Smartcard]
pin: false
mermaid: false
---

# Build guide: System Center Operations Manager 2025, from a single server to three

This is a step-by-step build guide, not a retrospective, each step is
something you actually do, in order, with a way to check it worked before
moving to the next one. It covers two builds in sequence, done as one
continuous effort on this environment: standing up SCOM 2025 as the
monitoring plane for a mixed environment (domain controllers, a SQL Always
On Availability Group, an MECM site, an RDS farm, and supporting
infrastructure) on a single management server first, agents to dashboards,
then scaling that single server out to three, with real agent load
distribution, confirmed notification resilience, and a load-balanced web
console that doesn't break PKI/smartcard authentication.

Follow it top to bottom on your own environment, substituting your own
hostnames, domain, and group names for this lab's (`myhomelab.hv.lab`,
`SUPERLAB`, `OPSMGR01`/`02`/`03`, etc.).

**End state**: a 3-node SCOM 2025 management group monitoring the whole
environment, dashboards with real performance data, synthetic monitoring
for internal web apps, and a highly available web console at
`https://scom-console.myhomelab.hv.lab/OperationsManager/` that survives
any one management server going down without breaking smartcard sign-in.

## Before you start

You need, already working:

- Active Directory Domain Services + DNS.
- A SQL Server target for SCOM's two databases (OperationsManager and the
  Data Warehouse). A standalone instance is simpler; this build used an
  existing Always On Availability Group listener instead, call out where
  that changes things as you go.
- A Windows Server 2025 VM for the first management server, domain-joined,
  with local admin rights for your build account. Two more identical VMs,
  domain-joined, for Steps 15-18's HA work, already added to the same
  management group (`setup.exe /components:OMServer` against the same
  operational database, that part isn't covered here).
- **Run everything in this guide from a domain-joined machine on the same
  network as your management servers**, ideally one of them directly. This
  isn't optional polish: a non-domain-joined or network-isolated box has no
  Kerberos identity for the web console's Windows Integrated Auth to
  negotiate with, and no default route to the internal domain. Every step
  past the initial install assumes you're already past that problem.
- (Optional, for the MECM deployment path in Step 3) A working MECM site
  with a distribution point reachable by your target machines. If you don't
  have one, skip to Step 3b and install the agent by hand instead,
  everything from Step 4 onward is identical either way.
- Product keys for anything you plan to license, and low expectations about
  how easy applying them will be, see Step 12.
- For Steps 15-18: an internal CA if you want the web console over HTTPS
  with a real certificate (self-signed works too, just adjust the cert
  steps), and a firewall/router that can host a TCP-mode reverse proxy,
  this build uses pfSense + the HAProxy package.

---


## Step 1: Install SCOM 2025

This is the standard Microsoft setup wizard; if you've installed any recent
SCOM version before, nothing here is conceptually different. Two things
below are worth getting right up front rather than retrofitting later: the
IIS prerequisites for the Web Console (step 2, the wizard's own error if
you skip it doesn't point at the real gap), and the management server's own
certificate (step 5).

**This assumes your SQL target, standalone instance or Availability
Group, is already built, healthy, and reachable.** Setting up or
hardening the SQL tier itself (AG configuration, listener certificates,
encryption) is out of scope here.

1. On the management server, install prerequisites: .NET, the ODBC driver
   and OLE DB provider for SQL Server, and, if you want reports,
   either SQL Server Reporting Services or Power BI Report Server reachable
   from this box.

2. **IIS, if you're installing the Web Console component on this server**:
   easy to miss, since the core Management Server and Operations Console
   roles don't need it, but the Setup wizard's Web Console page hard-blocks
   on it if it's missing:

   ```powershell
   Install-WindowsFeature -Name Web-Server, Web-Common-Http, Web-Static-Content, `
     Web-Default-Doc, Web-Http-Errors, Web-Http-Logging, Web-Request-Monitor, `
     Web-Filtering, Web-Stat-Compression, Web-Mgmt-Console, `
     Web-Asp-Net45, Web-Net-Ext45, Web-ISAPI-Ext, Web-ISAPI-Filter, `
     Web-Windows-Auth, NET-WCF-HTTP-Activation45 -IncludeManagementTools
   ```

   The one that's easy to miss even in the GUI path: **WCF HTTP
   Activation**, filed under Server Manager's **Features → .NET Framework
   4.8 Features → WCF Services**, not under the IIS role itself. Its
   absence produces the exact same generic prerequisite-check failure as
   missing IIS entirely; nothing in the Setup wizard names it directly.

3. Run `Setup.exe` from the installation media and choose **Install** →
   select **Management server** and **Operations console** (and
   **Web console**, if step 2's prerequisites are in place, and
   **Reporting server** if applicable).

4. On the database configuration screens, point both the OperationsManager
   database and the Data Warehouse database at your SQL target. If your
   target is an Availability Group, two things are worth confirming before
   you click through this screen:
   - Point setup at the **AG listener name**, not a node name.
   - Make sure the listener already has a working SPN
     (`MSSQLSvc/<listener-fqdn>:1433`) registered, or the initial connection
     during setup will fail with a generic connectivity error that doesn't
     mention Kerberos at all.

5. **Request the management server's own certificate before you get to Step
   5's web console work**: it's easy to treat this as a later problem, but
   it's simplest to get right while you're already thinking about
   certificates. This is a *different* certificate from anything used so
   far, and it deserves its own explanation, because the naming on it looks
   wrong at first glance and isn't.

   The management server's computer name in this build is `OPSMGR01`, but
   the certificate bound to its web console is issued to
   `opsmgr.myhomelab.hv.lab`, a shorter, friendlier name, not the computer's
   own name:

   ```powershell
   Get-ChildItem Cert:\LocalMachine\My | Select-Object Subject, Thumbprint, Issuer
   # Subject    : CN=opsmgr.myhomelab.hv.lab
   # Thumbprint : 6A1DA273375ED9C628360B0487EBDA5ED9DB396C
   # Issuer     : CN=myhomelab-DC01-CA, DC=myhomelab, DC=hv, DC=lab
   ```

   **This is deliberate, and it's a single-purpose certificate**: it exists
   *only* for the IIS web console binding you'll set up in Step 5, not for
   agent traffic. Two things make that possible:

   - `opsmgr.myhomelab.hv.lab` is a second DNS A record pointing at the same
     IP as `opsmgr01.myhomelab.hv.lab`, not a CNAME, a plain second A
     record, created purely so the console has a clean, memorable URL
     (`https://opsmgr.myhomelab.hv.lab/OperationsManager/`) independent of
     the underlying computer name.
   - Agent-to-management-server traffic (port 5723, the one you'll point
     `MANAGEMENT_SERVER_DNS` at back in Step 3) doesn't use this
     certificate, or IIS, or TLS at all in the way a website does; it's
     SCOM's own mutual-authentication channel, keyed off the computer's
     real identity (`OPSMGR01`). **Don't try to make this certificate cover
     both names "to be safe"; the two are unrelated by design, and giving
     the web cert the computer's own name too wouldn't change how agents
     authenticate.**

   Because the certificate's name needs to be something *other than* the
   requesting computer's own identity, the standard `WebServer` template
   won't work as-is: by default it auto-fills the subject/SAN from the
   requesting computer's own AD identity and won't let you override it.
   This build published a copy of the template with "supply in the request"
   enabled for the subject name, named it `WebServerManualSAN` to make the
   distinction obvious in the template list, and requested against that:

   ```powershell
   $infPath = 'C:\opsmgr-cert-request.inf'
   @'
[Version]
Signature="$Windows NT$"

[NewRequest]
Subject = "CN=opsmgr.myhomelab.hv.lab"
KeySpec = 1
KeyLength = 2048
Exportable = TRUE
MachineKeySet = TRUE
SMIME = FALSE
PrivateKeyArchive = FALSE
UserProtected = FALSE
UseExistingKeySet = FALSE
ProviderName = "Microsoft RSA SChannel Cryptographic Provider"
ProviderType = 12
RequestType = PKCS10
KeyUsage = 0xa0

[Extensions]
2.5.29.17 = "{text}"
_continue_ = "dns=opsmgr.myhomelab.hv.lab&"

[RequestAttributes]
CertificateTemplate = WebServerManualSAN
'@ | Set-Content -Path $infPath

   certreq -new $infPath 'C:\opsmgr-cert-request.req'
   certreq -submit -config '-' 'C:\opsmgr-cert-request.req' 'C:\opsmgr-cert.cer'
   certreq -accept 'C:\opsmgr-cert.cer'
   ```

   `certreq -submit -config '-'` prompts you to pick the CA interactively;
   pass `-config "<ca-server>\<ca-name>"` instead if you want it
   non-interactive. Once accepted, the cert lands in
   `Cert:\LocalMachine\My` ready for the IIS binding work in Step 5.

6. Give the management group a name (this build used `SUPERLAB`); you
   cannot change this later without a full reinstall, so pick something you
   won't mind seeing in every console screen indefinitely.
7. Accept the default Local System / computer account for the management
   server's action account unless you have a specific reason not to; this
   build left it at the default.
8. Finish the wizard, then confirm the management group is up:

   ```powershell
   Import-Module OperationsManager
   New-SCOMManagementGroupConnection -ComputerName localhost
   Get-SCOMManagementGroup | Select-Object Name
   ```

   You should get your management group name back with no errors. If this
   fails, stop here; nothing past this point will work until it's clean.

**Firewall**: SCOM Setup automatically adds the Windows Firewall rule for
agent traffic (TCP 5723) on the management server as part of installing
the Management Server role; no manual rule needed on a default firewall
setup. Worth confirming it's actually there if this box's firewall policy
is GPO-managed:

```powershell
Get-NetFirewallRule -DisplayGroup "System Center Operations Manager" | Select-Object DisplayName, Enabled
```

This doesn't cover everything, though: outbound 5723 from each *agent*
(hardened baselines and third-party host firewalls can silently block it,
looking identical to any other connectivity problem in Step 4), port 443
for the web console (normally auto-enabled by the IIS role install, but
confirm rather than assume), and port 445/SMB for the `SCOMAgent$` share
in Step 2 below (the "File and Printer Sharing" rule group, off by default
on a server that's never shared a folder before) are all worth a quick
check rather than an assumption, especially on a stricter environment than
this lab's default-Windows-Firewall setup.

---

## Step 2: Get the agent installer onto a share

The agent MSI (`MOMAgent.msi`) ships in the SCOM installation media under
`\Setup\AMD64\` (for x64 targets). Share it out from the management server so
targets can pull it:

```powershell
New-Item -Path 'C:\SCOMAgent' -ItemType Directory -Force
Copy-Item -Path '<install-media>\Setup\AMD64\MOMAgent.msi' -Destination 'C:\SCOMAgent\'
New-SmbShare -Name 'SCOMAgent$' -Path 'C:\SCOMAgent' -ReadAccess 'Domain Computers'
```

The `$` makes it a hidden share; targets can still reach it by full UNC
path, it just won't show up in casual network browsing.

---

## Step 3: Deploy the agent

Two ways to do this. Use 3a if you have MECM already; use 3b if you don't or
just want to install one agent by hand to test.

### Step 3a: Via MECM (the way that scales to a whole fleet)

Create an Application in the MECM console (or via PowerShell) with a single
Script Deployment Type whose install command is a **direct, explicit**
`msiexec` call, deliberately not using AD-based configuration, for reasons
covered below:

```
msiexec /i MOMAgent.msi /qn USE_SETTINGS_FROM_AD=0 USE_MANUALLY_SPECIFIED_SETTINGS=1 MANAGEMENT_GROUP=SUPERLAB MANAGEMENT_SERVER_DNS=opsmgr01.myhomelab.hv.lab SECURE_PORT=5723 ACTIONS_USE_COMPUTER_ACCOUNT=1 AcceptEndUserLicenseAgreement=1
```

Replace `SUPERLAB`, `opsmgr01.myhomelab.hv.lab`, and the port if you changed
it from the default `5723`.

**Two settings in that line are not optional, and both fail silently if you
skip them:**

- **Detection method: use the MSI Product Code**, not a PowerShell-script
  detection method. Get the code from the MSI itself before you build the
  deployment type:

  ```powershell
  $wi = New-Object -ComObject WindowsInstaller.Installer
  $db = $wi.GetType().InvokeMember('OpenDatabase','InvokeMethod',$null,$wi,@('C:\SCOMAgent\MOMAgent.msi',0))
  $view = $db.GetType().InvokeMember('OpenView','InvokeMethod',$null,$db,@("SELECT Value FROM Property WHERE Property='ProductCode'"))
  $view.GetType().InvokeMember('Execute','InvokeMethod',$null,$view,$null)
  $record = $view.GetType().InvokeMember('Fetch','InvokeMethod',$null,$view,$null)
  $record.GetType().InvokeMember('StringData','GetProperty',$null,$record,1)
  ```

  A script-based detection method will look reasonable and match other
  deployment types in the same MECM site, and will fail outright the moment
  a target's PowerShell execution policy blocks it, before it ever gets a
  chance to check whether the agent is present.

- **`USE_MANUALLY_SPECIFIED_SETTINGS=1` must be present alongside
  `USE_SETTINGS_FROM_AD=0`.** Skip it and the agent installs, HealthService
  starts, and then stops itself within seconds; check the target's
  Operations Manager event log and you'll see event 2001 ("No Management
  Group could be started") followed by 2017 ("Active Directory integration
  has been disabled"). Setting `USE_SETTINGS_FROM_AD=0` alone tells the
  agent not to look in AD; without the second flag, nothing tells it to use
  the server/port you just gave it on the command line either, and it ends
  up with no configuration source at all.

Target every server you want monitored with this deployment, **domain
controllers included**. The alternative (AD-based assignment,
`USE_SETTINGS_FROM_AD=1`) looks like the more "correct" design and is worth
knowing about, but has two dealbreakers: it fails outright on domain
controllers (SCOM force-disables AD Integration on any Health Service
running on a DC, event 2119, not configurable), and it depends on SCOM's AD
Assignment Resource Pool successfully writing connection data into an AD
Service Connection Point, which in this build never happened despite
correct-looking permissions. If you want to chase AD-based assignment
anyway, confirm the SCP write is actually landing
(`CN=HealthServiceSCP,CN=<MgmtGroup>,CN=OperationsManager,<domain-DN>`,
check its `serviceBindingInformation` attribute is non-empty) before assuming
a permissions grant fixed it: an empty attribute means it didn't, no matter
what the grant looked like.

### Step 3b: By hand, one machine at a time

```powershell
$args = @(
    '/i', 'C:\SCOMAgent\MOMAgent.msi', '/qn',
    'USE_SETTINGS_FROM_AD=0', 'USE_MANUALLY_SPECIFIED_SETTINGS=1',
    'MANAGEMENT_GROUP=SUPERLAB',
    'MANAGEMENT_SERVER_DNS=opsmgr01.myhomelab.hv.lab',
    'SECURE_PORT=5723', 'ACTIONS_USE_COMPUTER_ACCOUNT=1',
    'AcceptEndUserLicenseAgreement=1'
) -join ' '
Start-Process msiexec.exe -ArgumentList $args -Wait
Get-Service HealthService | Select-Object Status
```

`Status` should read `Running` and stay running; recheck it after 30
seconds, since the "installs then immediately dies" failure mode from the
missing flag above takes a few seconds to show itself.

---

## Step 4: Verify agents are actually talking to the management server

Local `HealthService: Running` is not proof the agent is connected. Verify
from the management server side too:

```powershell
Import-Module OperationsManager
New-SCOMManagementGroupConnection -ComputerName localhost

# Newly-installed agents land here first, waiting for approval
Get-SCOMPendingManagement | Select-Object AgentName, AgentPendingActionType

# Approve everything currently pending
Get-SCOMPendingManagement | Approve-SCOMPendingManagement

# Confirm registration
Get-SCOMAgent | Select-Object DisplayName, HealthState
```

If a freshly-deployed agent never shows up in `Get-SCOMPendingManagement` at
all (not even as a rejected attempt), check the management group's manual
agent approval setting:

```powershell
Get-SCOMAgentApprovalSetting
```

If it's not set to review/approve manually-installed agents, new agents get
outright rejected before they ever reach the pending queue: the target's
own event log will show event 20000 ("A device which is not part of this
management group has attempted to access this Health Service") and 26321
("An agent was rejected. Current security settings do not allow the
automatic insertion of agents"). Fix with `Set-SCOMAgentApprovalSetting`.

---

## Step 5: Fix web console sign-in

This is the step most likely to eat an afternoon if you go in without a
checklist. Six separate things have to be true simultaneously for Windows
Integrated Auth to the web console to work from a domain-joined machine;
missing any one of them looks identical to missing all of them (a login
failure or endless prompt), so work through all six rather than trying one
and assuming failure means it was the wrong one.

1. **Local Intranet zone**, via Group Policy, for the console's hostname
   (`ZoneMap\Domains\<domain>\<console-hostname>` = zone 1, both `http` and
   `https` values).
2. **Edge's own SSO allowlist**: Chromium Edge maintains a separate
   Kerberos allowlist from the classic IE zone map. You need the
   `msedge.admx`/`.adml` templates in your GPO Central Store, then set
   `AuthServerAllowlist` and `AuthNegotiateDelegateAllowlist` (both under
   `Policies\Microsoft\Edge`) to the console's hostname.
3. **NTLM loopback exemption**, if you're testing directly on the management
   server itself: add the console's hostname(s) to
   `HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\BackConnectionHostNames`
   (multi-string value, one hostname per line); without this, local
   loopback calls to the console over a name (not `localhost`) get a 401
   from the NTLM loopback check regardless of credentials.
4. **`IIS_IUSRS` needs read on `redirection.config`**:
   ```powershell
   icacls 'C:\Windows\System32\inetsrv\config\redirection.config' /grant 'IIS_IUSRS:(R)'
   ```
   Without it, the web console's data service can't do WCF service
   discovery, and the failure surfaces as a generic, unhelpful error with
   nothing in the Application event log pointing at the real cause.
5. **`BUILTIN\Administrators` needs to be a member of the Operations Manager
   Administrators role.** If you've been hardening this role by removing
   built-in groups in favor of named accounts, know that it breaks the web
   console's own service-account discovery.
6. **IIS bindings, the one that actually mattered in this build.** An HTTPS
   binding created with a specific hostname and SNI enabled will only
   complete a TLS handshake for connections presenting that exact SNI name.
   Internal loopback calls the console's own backend makes to itself
   (targeting `localhost`, not the public hostname) fail the handshake
   before authentication is even in play. Fix with *both* of the following;
   either alone is insufficient:
   ```powershell
   # Default (non-SNI) certificate binding, catches "localhost" and anything else -
   # this is the same opsmgr.myhomelab.hv.lab certificate requested back in Step 1
   netsh http add sslcert ipport=0.0.0.0:443 certhash=6A1DA273375ED9C628360B0487EBDA5ED9DB396C appid='{00000000-0000-0000-0000-000000000000}'
   ```
   ```powershell
   # Blank-hostheader HTTPS site binding, for host-header-based routing
   New-WebBinding -Name 'Default Web Site' -Protocol https -Port 443 -HostHeader '' -SslFlags 0
   ```

**If login is still failing after all six**, check the IIS bindings (#6)
again first: it's the one most likely to look fine on a casual check
(the site *has* an HTTPS binding, after all) while still being the actual
blocker.

### Turning on deeper IIS diagnostics, for when none of the six explain it

Fix #4 above is the failure mode you'll hit most: a generic browser error
with nothing in the Application event log pointing at the real cause.
Two built-in IIS tools get you past that wall without guessing.

**W3C logging with the auth-specific fields**, first: free, always-on
once configured:

```powershell
Set-WebConfiguration -Filter "system.applicationHost/sites/siteDefaults/logFile" -Value @{
    logExtFileFlags = "Date,Time,ClientIP,UserName,Method,UriStem,UriQuery,HttpStatus,HttpSubStatus,Win32Status,TimeTaken"
}
```

Reproduce the failing sign-in, then read the `sc-status`/`sc-substatus`/
`sc-win32-status` triplet in the newest file under
`C:\inetpub\logs\LogFiles\W3SVC1\`: `401.1` means a genuine credential
problem (downstream of all six fixes above, not a config gap); `401.2`
means Windows Authentication isn't enabled or negotiate is failing
entirely (recheck fix #5); `401.5` means the web console's own WCF handler
rejected the request (recheck fix #4 and fix #6).

**Failed Request Tracing (FREB)**, if the status/substatus table still
doesn't explain it or the failure is a 500 rather than a 401: captures a
full step-by-step trace of the request through every IIS/ASP.NET module:

```powershell
Install-WindowsFeature -Name Web-Http-Tracing

& "$env:windir\system32\inetsrv\appcmd.exe" set config "Default Web Site" `
  -section:system.applicationHost/sites `
  "/[name='Default Web Site'].traceFailedRequestsLogging.enabled:true" /commit:apphost

& "$env:windir\system32\inetsrv\appcmd.exe" set config "Default Web Site/OperationsManager" `
  -section:system.webServer/tracing/traceFailedRequests `
  "/+[path='*',statusCodes='400-999']" /commit:apphost
```

Reproduce the failure, then open the newest `.xml` file under
`C:\inetpub\logs\FailedReqLogFiles\W3SVC1\` **in a browser, not a text
editor**: IIS drops a `freb.xsl` stylesheet in the same folder that
renders it as a readable request timeline. Turn both diagnostics off again
once you've found the problem; FREB especially writes a file per failed
request and isn't something to leave running against real traffic.

---

## Step 6: Fix a role-membership gotcha before you need it

Separately from console login: certain privileged SCOM cmdlets (licensing
operations are the ones that bit this build) require the calling account to
be a **direct** member of Operations Manager Administrators; membership
through a nested group doesn't count for these specific operations, even
though it's entirely sufficient for normal console access.

```powershell
$role = Get-SCOMUserRole -Name 'OperationsManagerAdministrators'
$isDirect = $role.Users -contains 'YOURDOMAIN\your-account'
if (-not $isDirect) {
    Set-SCOMUserRole -UserRole $role -User ($role.Users + 'YOURDOMAIN\your-account')
}
```

Do this now, before you hit a cmdlet that fails for no apparent reason and
burns time on the wrong hypothesis.

---

## Step 7: Build functional groups

Out of the box, the console shows every monitored computer as one flat
list. This script builds seven role-based groups, each a singleton
`ComputerGroup` populated by a regex match against the computer's FQDN, and
bakes in one more fix you need regardless of how you get there (explained
below the script). Adjust `$groups` to your own hostname conventions and
save it as `Build-Groups.ps1`:

```powershell
Import-Module OperationsManager -ErrorAction Stop
New-SCOMManagementGroupConnection -ComputerName OPSMGR01.myhomelab.hv.lab -ErrorAction Stop

$groups = @(
    @{ Short = "DomainControllers";  DisplayName = "SUPERLAB - Domain Controllers";  Pattern = '^DC\d+\..*' }
    @{ Short = "SQLServers";         DisplayName = "SUPERLAB - SQL Servers";         Pattern = '^SQL\d+\..*' }
    @{ Short = "MECMServers";        DisplayName = "SUPERLAB - MECM Servers";        Pattern = '^MECM\d+\..*' }
    @{ Short = "WSUSServers";        DisplayName = "SUPERLAB - WSUS Servers";        Pattern = '^WSUS\d+\..*' }
    @{ Short = "RDSFarm";            DisplayName = "SUPERLAB - RDS Farm";            Pattern = '^RDS.*\..*' }
    @{ Short = "CoreInfrastructure"; DisplayName = "SUPERLAB - Core Infrastructure"; Pattern = '^(DHCP01|FS01|NPS01)\..*' }
)

$classTypes = New-Object System.Collections.Generic.List[string]
$discoveries = New-Object System.Collections.Generic.List[string]
$displayStrings = New-Object System.Collections.Generic.List[string]

foreach ($g in $groups) {
    $classId = "SUPERLAB.Group.$($g.Short)"
    $classTypes.Add("        <ClassType ID=`"$classId`" Accessibility=`"Public`" Abstract=`"false`" Base=`"SC!Microsoft.SystemCenter.ComputerGroup`" Hosted=`"false`" Singleton=`"true`" Extension=`"false`" />")
    $discoveries.Add(@"
      <Discovery ID="$classId.Discovery" Enabled="true" Target="$classId" ConfirmDelivery="false" Remotable="true" Priority="Normal">
        <Category>Discovery</Category>
        <DiscoveryTypes><DiscoveryClass TypeID="$classId" /></DiscoveryTypes>
        <DataSource ID="DS" TypeID="SC!Microsoft.SystemCenter.GroupPopulator">
          <RuleId>`$MPElement`$</RuleId>
          <GroupInstanceId>`$MPElement[Name="$classId"]`$</GroupInstanceId>
          <MembershipRules>
            <MembershipRule>
              <MonitoringClass>`$MPElement[Name="Windows!Microsoft.Windows.Computer"]`$</MonitoringClass>
              <RelationshipClass>`$MPElement[Name="SC!Microsoft.SystemCenter.ComputerGroupContainsComputer"]`$</RelationshipClass>
              <Expression>
                <RegExExpression>
                  <ValueExpression><Property>`$MPElement[Name="Windows!Microsoft.Windows.Computer"]/PrincipalName`$</Property></ValueExpression>
                  <Operator>MatchesRegularExpression</Operator>
                  <Pattern>$($g.Pattern)</Pattern>
                </RegExExpression>
              </Expression>
            </MembershipRule>
          </MembershipRules>
        </DataSource>
      </Discovery>
"@)
    $displayStrings.Add("        <DisplayString ElementID=`"$classId`"><Name>$($g.DisplayName)</Name></DisplayString>")
}

$mpXml = @"
<?xml version="1.0" encoding="utf-8"?>
<ManagementPack ContentReadable="true" SchemaVersion="2.0" OriginalSchemaVersion="1.1">
  <Manifest>
    <Identity><ID>SUPERLAB.Groups</ID><Version>1.0.0.0</Version></Identity>
    <Name>SUPERLAB Functional Groups</Name>
    <References>
      <Reference Alias="Windows"><ID>Microsoft.Windows.Library</ID><Version>7.5.8501.1</Version><PublicKeyToken>31bf3856ad364e35</PublicKeyToken></Reference>
      <Reference Alias="SCLibrary"><ID>System.Library</ID><Version>7.5.8501.1</Version><PublicKeyToken>31bf3856ad364e35</PublicKeyToken></Reference>
      <Reference Alias="SC"><ID>Microsoft.SystemCenter.Library</ID><Version>10.25.10132.0</Version><PublicKeyToken>31bf3856ad364e35</PublicKeyToken></Reference>
      <Reference Alias="Windows1"><ID>Microsoft.Windows.Server.2016.Discovery</ID><Version>10.1.2.2</Version><PublicKeyToken>31bf3856ad364e35</PublicKeyToken></Reference>
    </References>
  </Manifest>
  <TypeDefinitions><EntityTypes><ClassTypes>
$($classTypes -join "`n")
  </ClassTypes></EntityTypes></TypeDefinitions>
  <Monitoring>
    <Discoveries>
$($discoveries -join "`n")
    </Discoveries>
    <Overrides>
      <!-- Without this, Processor/Memory/LogicalDisk performance counters never populate for
           ANY agent, no matter how many role-specific MPs you import on top - see the note
           below the script for why this is here. -->
      <DiscoveryPropertyOverride ID="Override.EnableWinServerCpuDiscovery" Context="Windows1!Microsoft.Windows.Server.10.0.OperatingSystem" Enforced="false" Discovery="Windows1!Microsoft.Windows.Server.10.0.CPU.Discovery" Property="Enabled">
        <Value>true</Value>
      </DiscoveryPropertyOverride>
    </Overrides>
  </Monitoring>
  <LanguagePacks><LanguagePack ID="ENU" IsDefault="true"><DisplayStrings>
    <DisplayString ElementID="SUPERLAB.Groups"><Name>SUPERLAB Functional Groups</Name><Description>Functional server groups, organized by role.</Description></DisplayString>
$($displayStrings -join "`n")
  </DisplayStrings></LanguagePack></LanguagePacks>
</ManagementPack>
"@

$outPath = "C:\SCOMBuild\SUPERLAB.Groups.xml"
New-Item -Path (Split-Path $outPath) -ItemType Directory -Force | Out-Null
[System.IO.File]::WriteAllText($outPath, $mpXml, (New-Object System.Text.UTF8Encoding($false)))
[xml]$validate = Get-Content $outPath -Raw   # throws here if the XML isn't well-formed
Write-Host "Wrote $outPath - $($validate.ManagementPack.TypeDefinitions.EntityTypes.ClassTypes.ClassType.Count) groups"

Import-SCOMManagementPack -Fullname $outPath
```

**Why that override is in there**: before you build a single performance
dashboard widget, check whether performance data actually exists.
Processor/Memory/Disk counters depend on the
`Microsoft.Windows.Server.OperatingSystem` class being discovered, and its
CPU discovery rule is **disabled by default**. Without the override above,
every one of your agents will show a health state but zero performance
history, and every trend chart you build in Step 9 will be empty, not
broken, just genuinely never going to have data. Confirm it worked:

```powershell
(Get-SCOMClassInstance -Class (Get-SCOMClass -Name 'Microsoft.Windows.Server.OperatingSystem')).Count
```

Zero means the override didn't take (or needs more time to propagate; give
it one discovery cycle, typically under an hour, before troubleshooting
further). Non-zero and growing toward your agent count means it worked.

### CPU isn't the only discovery worth auditing

That single override fixes Processor data specifically; it doesn't mean
every other discovery you'll want is already on. A useful habit before you
build *any* dashboard widget: check what's actually disabled, for the class
you're about to build against, rather than assuming a management pack
ships with everything on.

```powershell
Get-SCOMDiscovery -Name '*Windows.Server.10.0*' | Select-Object DisplayName, Enabled | Sort-Object Enabled
```

Run against this build's Windows Server management pack, it turned up
several more disabled-by-default discoveries beyond CPU: **Windows Disk
Partitions**, **Windows Physical Disks**, **Mount Points**, and **Network
Adapters (Both Enabled and Disabled)**, the last one has an *enabled*
sibling rule, "Network Adapters (Only Enabled)," which is why it's easy to
assume adapter discovery is already covered when only half of it is.
Logical Disk discovery, by contrast, is already on by default, which is
why the dashboard's disk-space widget in Step 9 doesn't need an override at
all. **Not every management pack needs this treatment**: running the same
query against the IIS management pack, every single discovery in it ships
enabled by default. Check before you assume either way.

Once you've found one you need, you have the same two options as the CPU
fix: hand-author another `DiscoveryPropertyOverride` block like the one
above, or let the console build the override for you:

- **Console GUI**: Authoring workspace → Management Pack Objects → Object
  Discoveries. Change the "Scope" filter (top right) to your target class if
  the list is too long to scan. Right-click the discovery you want →
  **Overrides → Override the Object Discovery → For all objects of class:
  \<your class\>**. In the override properties, check **Enabled**, set the
  value to **True**, and, same rule as everywhere else in this build, save
  it to a *new* unsealed management pack rather than the default "always
  save to" one, so your override survives if the original MP ever gets
  re-imported at a newer version.
- **Scripted**: same technique as the CPU override, a
  `DiscoveryPropertyOverride` element, `Context` set to the class the
  discovery targets, `Discovery` set to the discovery's own ID, `Property`
  set to `Enabled`, `Value` set to `true`. Copy the pattern from the
  `SUPERLAB.Groups` MP above and point it at whichever discovery you found.

**Console-tree tip while you're in here**: if you also build custom Views
for these groups, parent their folder under
`SC!Microsoft.SystemCenter.Monitoring.ViewFolder.Root`, not
`Microsoft.SystemCenter.ViewFolder.Root`; the second one imports without
error but sits above the Monitoring workspace tab itself, so a folder
parented there never appears as a visible tree item in either console. Easy
mistake, invisible until you go looking for the folder and can't find it.

---

## Step 8: Import role-specific management packs

Generic OS health only gets you so far. Import the Microsoft-catalog MP for
every role actually running in your environment: this build used IIS,
Windows Failover Clustering, DHCP, AD CS, AD DS, and SQL Server. Download
each from the Microsoft Download Center, extract the `.msi` (they're
self-extracting), then:

```powershell
$files = Get-ChildItem "<extracted-folder>\*.mp","<extracted-folder>\*.mpb" | Select-Object -ExpandProperty FullName
Import-SCOMManagementPack -Fullname $files
```

`Import-SCOMManagementPack` accepts an array and resolves dependency order
within that batch automatically; you don't need to hand-sequence "library
before discovery before monitoring," as long as every file a given batch
depends on is either already imported or included in the same call.

**Check for clustering before you skip the Cluster MP.** If your SQL tier
sits on an Always On Availability Group, the nodes are very likely also
Windows Failover Cluster members even if nobody's mentioned it explicitly;
confirm with:

```powershell
(Get-SCOMClassInstance -Class (Get-SCOMClass -Name 'Microsoft.Windows.Cluster.Node')).Count
```

Non-zero means import the Cluster MP too, even if your SQL role MP alone
looked sufficient on paper.

---

## Step 9: Build a dashboard

The console's Dashboard Designer is the "normal" way to build dashboards,
but dashboard content, despite persistent community folklore that it's
GUI-only, imports through `Import-SCOMManagementPack` exactly like any
other management pack, using the HTML5 dashboard framework
(`Microsoft.SystemCenter.HTMLDashboardViewType` /
`...HTMLWidgetType`). That means dashboards can be versioned, generated in a
loop, and code-reviewed instead of being one-off console clickwork, worth
doing this way even for a single dashboard, and essential once you're
building more than one.

Below is one complete, working dashboard: a state rollup, a memory trend
chart, and an alerts list, scoped to one group from Step 7. Get your group's
internal ID first:

```powershell
$group = Get-SCOMClassInstance -Class (Get-SCOMClass -Name 'SUPERLAB.Group.DomainControllers')
$groupId = $group.Id.Guid
```

Then:

```powershell
$scopeJson = "{`"scopeSelection`":[{`"id`":`"$groupId`",`"displayName`":`"SUPERLAB - Domain Controllers`",`"className`":`"SUPERLAB - Domain Controllers`",`"path`":null,`"fullName`":`"SUPERLAB.Group.DomainControllers`",`"objectType`":-1}]}"

$stateConfig  = "{`"widgetDisplay`":{`"col`":1,`"row`":1,`"sizex`":12,`"sizey`":5,`"columns`":[`"healthstate`",`"displayname`"],`"payload`":`"$([guid]::NewGuid())`",`"dragHandle`":`".draggable`",`"resizeHandle`":`".resizable`"},`"widgetParameters`":{`"scope`":$scopeJson,`"criteria`":{`"healthStates`":[`"2`",`"0`",`"1`",`"3`"],`"inMaintenanceMode`":`"All`"}},`"widgetRefreshInterval`":5}"
$memConfig    = "{`"widgetDisplay`":{`"col`":1,`"row`":7,`"sizex`":12,`"sizey`":5,`"selectedLegends`":[`"Path`",`"AverageValue`"],`"visualizeObjectsByPerformance`":true,`"payload`":`"$([guid]::NewGuid())`",`"dragHandle`":`".draggable`",`"resizeHandle`":`".resizable`"},`"widgetParameters`":{`"scope`":$scopeJson,`"criteria`":{`"timeRange`":{`"timeValue`":24,`"timeUnit`":`"Hours`"},`"performanceCounters`":[{`"objectname`":`"Memory`",`"countername`":`"Available MBytes`",`"instancename`":`"`"}]}},`"widgetRefreshInterval`":5}"
$alertsConfig = "{`"widgetDisplay`":{`"col`":1,`"row`":13,`"sizex`":12,`"sizey`":5,`"columns`":[`"severity`",`"monitoringobjectdisplayname`",`"name`",`"age`"],`"groupByColumn`":`"severity`",`"payload`":`"$([guid]::NewGuid())`",`"dragHandle`":`".draggable`",`"resizeHandle`":`".resizable`"},`"widgetParameters`":{`"scope`":$scopeJson,`"criteria`":{`"severities`":[`"1`",`"2`"],`"priorities`":[`"2`",`"1`"],`"resolutionStates`":[`"0`"],`"age`":7,`"ageTimeUnit`":`"days`"}},`"widgetRefreshInterval`":5}"

$mpXml = @"
<?xml version="1.0" encoding="utf-8"?>
<ManagementPack ContentReadable="true" SchemaVersion="2.0" OriginalSchemaVersion="1.1">
  <Manifest>
    <Identity><ID>SUPERLAB.Dashboards.DomainControllers</ID><Version>1.0.0.0</Version></Identity>
    <Name>SUPERLAB Domain Controllers Dashboard</Name>
    <References>
      <Reference Alias="SCInternal"><ID>Microsoft.SystemCenter.Visualization.Library</ID><Version>10.25.10132.0</Version><PublicKeyToken>31bf3856ad364e35</PublicKeyToken></Reference>
    </References>
  </Manifest>
  <Monitoring>
    <Views>
      <View ID="SUPERLAB.Dashboard.DC.State" Accessibility="Public" Enabled="true" Target="System!System.Entity" TypeID="SCInternal!Microsoft.SystemCenter.HTMLWidgetType" Visible="true">
        <Category>Operations</Category>
        <WidgetConfiguration><Configuration>$stateConfig</Configuration><Type>HtmlStateWidget</Type></WidgetConfiguration>
      </View>
      <View ID="SUPERLAB.Dashboard.DC.Memory" Accessibility="Public" Enabled="true" Target="System!System.Entity" TypeID="SCInternal!Microsoft.SystemCenter.HTMLWidgetType" Visible="true">
        <Category>Operations</Category>
        <WidgetConfiguration><Configuration>$memConfig</Configuration><Type>HtmlPerformanceWidget</Type></WidgetConfiguration>
      </View>
      <View ID="SUPERLAB.Dashboard.DC.Alerts" Accessibility="Public" Enabled="true" Target="System!System.Entity" TypeID="SCInternal!Microsoft.SystemCenter.HTMLWidgetType" Visible="true">
        <Category>Operations</Category>
        <WidgetConfiguration><Configuration>$alertsConfig</Configuration><Type>HtmlAlertWidget</Type></WidgetConfiguration>
      </View>
      <View ID="SUPERLAB.Dashboard.DC.Container" Accessibility="Public" Enabled="true" Target="System!System.Entity" TypeID="SCInternal!Microsoft.SystemCenter.HTMLDashboardViewType" Visible="true">
        <Category>Operations</Category>
        <DashboardConfiguration>
          <Configuration />
          <Type />
          <Widgets>
            <Widget>SUPERLAB.Dashboard.DC.State</Widget>
            <Widget>SUPERLAB.Dashboard.DC.Memory</Widget>
            <Widget>SUPERLAB.Dashboard.DC.Alerts</Widget>
          </Widgets>
        </DashboardConfiguration>
      </View>
    </Views>
  </Monitoring>
  <LanguagePacks><LanguagePack ID="ENU" IsDefault="true"><DisplayStrings>
    <DisplayString ElementID="SUPERLAB.Dashboards.DomainControllers"><Name>SUPERLAB Domain Controllers Dashboard</Name></DisplayString>
    <DisplayString ElementID="SUPERLAB.Dashboard.DC.State"><Name>Domain Controllers - State</Name></DisplayString>
    <DisplayString ElementID="SUPERLAB.Dashboard.DC.Memory"><Name>Domain Controllers - Available Memory</Name></DisplayString>
    <DisplayString ElementID="SUPERLAB.Dashboard.DC.Alerts"><Name>Domain Controllers - Active Alerts</Name></DisplayString>
    <DisplayString ElementID="SUPERLAB.Dashboard.DC.Container"><Name>Domain Controllers - Dashboard</Name></DisplayString>
  </DisplayStrings></LanguagePack></LanguagePacks>
</ManagementPack>
"@

$outPath = "C:\SCOMBuild\SUPERLAB.Dashboards.DomainControllers.xml"
[System.IO.File]::WriteAllText($outPath, $mpXml, (New-Object System.Text.UTF8Encoding($false)))
Import-SCOMManagementPack -Fullname $outPath
```

Open the console and confirm the dashboard appears under Monitoring with a
populated state table and (after the first performance-collection cycle,
typically a few minutes) a memory trend line.

**To scale this to more dashboards**, wrap the widget-building block in a
loop over an array of `{ Short, DisplayName, GroupId }` hashtables, one per
group from Step 7, and give every widget ID and `<View ID>` a
group-specific suffix so they don't collide on import; that's the entire
technique, just repeated.

**Legend rendering bug to know about**: if you add `"Target"` to a
`selectedLegends` array, the console renders raw internal GUIDs (and
sometimes what look like timestamps) instead of server names on some rows.
This is a client-side rendering bug in how the `Target` column resolves a
caption when the view's own `Target` attribute is the generic
`System!System.Entity` (which every dashboard view here uses), not a data
problem. Leave `"Target"` out of `selectedLegends` and use `"Path"` instead,
as the example above already does.

---

## Step 10: SQL Server discovery: the gMSA trap

If you imported the SQL Server MP in Step 8 and `Get-SCOMClassInstance
-Class (Get-SCOMClass -Name 'Microsoft.SQLServer.Windows.DBEngine')` comes
back empty despite SQL Server genuinely running on your targets, this
section is why, and it's worth reading in full before you try the obvious
fix.

**Root cause**: the "Microsoft SQL Server Discovery Run As Profile" has no
account mapped. Without a credential, the discovery workflow has nothing to
run as, and it fails without logging an error you'd notice; it just never
produces results.

**Do not do this, even though it looks correct and returns no error:**

```powershell
# BROKEN - do not use for a gMSA
$cred = New-Object System.Management.Automation.PSCredential("MYDOMAIN\svc-scomsql$", (New-Object System.Security.SecureString))
Add-SCOMRunAsAccount -Name "SQL Discovery Account" -RunAsCredential $cred -Windows
```

`Add-SCOMRunAsAccount -Windows` has no gMSA-aware code path in this SCOM
version. An empty `SecureString` just stores a normal account credential
with a literal blank password. SCOM then tries to log the gMSA on with
that blank password (a gMSA's real password is a complex value AD manages,
that SCOM never learns through this path) and fails, repeating event 1108
("An Account specified in the Run As Profile ... cannot be resolved") every
~30 minutes indefinitely, with nothing more specific logged anywhere.

**The real fix needs the desktop Operations Console; there is no
PowerShell equivalent for this specific step:**

1. If you already created the broken account, remove it first:
   ```powershell
   $broken = Get-SCOMRunAsAccount -Name "SQL Discovery Account"
   Set-SCOMRunAsProfile -Action Remove -Profile (Get-SCOMRunAsProfile -DisplayName "Microsoft SQL Server Discovery Run As Profile") -Account $broken
   Remove-SCOMRunAsAccount -RunAsAccount $broken
   ```
2. In the Operations Console: **Administration → Run As Configuration →
   Accounts → Create Run As Account.**
3. Account type: **Windows**. Check **"This account is a Group Managed
   Service Account."** This checkbox is the entire fix; it does not exist
   in any cmdlet in this SCOM version.
4. Finish the wizard, pointing it at your gMSA.

Then, back in PowerShell, finish the setup:

> **⚠ This grants `sysadmin`, the highest privilege level SQL Server has, to
> a service account on every SQL instance you list below.** Microsoft's
> own Run As Profiles documentation for this management pack lists SA
> (sysadmin) rights as one of its normal, supported configurations, so
> this isn't a shortcut or a mistake, it's a legitimate path. It's still
> a broad, standing grant on production SQL instances though, worth
> being deliberate about rather than running on reflex. If your
> environment's security policy won't tolerate a service account
> holding `sysadmin` at all, Microsoft documents a separate low-privilege
> configuration for this exact management pack as the alternative for
> that case, not a quick follow-up cleanup step, a genuinely different
> setup: three separate domain accounts (one for discovery, one for
> monitoring, one for task execution), a dedicated low-privilege server
> role granted `VIEW SERVER STATE`, `VIEW ANY DEFINITION`, and `VIEW ANY
> DATABASE` plus a specific list of narrow stored-procedure `EXECUTE`
> grants (things like `sys.xp_readerrorlog`, `sys.xp_instance_regread`,
> and read access to the SQL Agent job tables in `msdb`), matching WMI
> namespace permissions on each target, and separate Run As Profile
> mappings for each of the three accounts. If your policy requires that
> path, budget real time for it and search Microsoft Learn for
> "low-privilege monitoring in Management Pack for SQL Server"; don't
> try to retrofit it by revoking `sysadmin` after the fact with nothing
> else in place to replace it. Know what you're granting before you run
> the command below either way, and scope the account list to exactly
> the instances this discovery actually needs, not every SQL Server you
> happen to manage. If you're running this build with any kind of
> automation or AI-agent assistance, expect this specific step to
> require a human hand on the keyboard, elevation grants like this one
> are exactly the category most agent safety guardrails are designed to
> stop cold, sometimes even after an explicit approval, see the note
> further down in this step.

```powershell
# Let the target agents retrieve the gMSA's managed password
Set-ADServiceAccount -Identity "svc-scomsql" -PrincipalsAllowedToRetrieveManagedPassword (
    "SQL01","SQL02","SQL03","OPSMGR01" | ForEach-Object { Get-ADComputer -Identity $_ }
)

# Grant it sysadmin on every SQL instance being discovered/monitored. Do this
# step interactively; see the warning above and the note below for why it may
# not run under automation
Invoke-Sqlcmd -ServerInstance "SQL01" -Query "CREATE LOGIN [MYDOMAIN\svc-scomsql`$] FROM WINDOWS; ALTER SERVER ROLE sysadmin ADD MEMBER [MYDOMAIN\svc-scomsql`$];"

# Associate the account with both SQL Run As profiles, scoped to the SQL nodes
$acct = Get-SCOMRunAsAccount -Name "SQL Discovery Account"
$sqlServers = "SQL01","SQL02","SQL03" | ForEach-Object {
    Get-SCOMClassInstance -Name "$_.myhomelab.hv.lab" | Where-Object FullName -like "Microsoft.Windows.Computer:*"
}
Set-SCOMRunAsProfile -Action Add -Profile (Get-SCOMRunAsProfile -DisplayName "Microsoft SQL Server Discovery Run As Profile") -Account $acct -Instance $sqlServers
Set-SCOMRunAsProfile -Action Add -Profile (Get-SCOMRunAsProfile -DisplayName "Microsoft SQL Server Monitoring Run As Profile") -Account $acct -Instance $sqlServers

# Force a fresh discovery attempt on each node
"SQL01","SQL02","SQL03" | ForEach-Object {
    Invoke-Command -ComputerName "$_.myhomelab.hv.lab" -ScriptBlock { Restart-Service HealthService -Force }
}
```

**Verify through the target's event log, not by polling
`Get-SCOMClassInstance`**: the class-instance count lags well behind the
actual fix:

```powershell
Get-WinEvent -LogName "Operations Manager" -ComputerName SQL01 | Where-Object Id -eq 7026   # Health Service logged on the account
Get-WinEvent -LogName "Operations Manager" -ComputerName SQL01 | Where-Object Id -eq 1109   # All credential references resolved
```

**Two more things to expect, not troubleshoot:**

- The seed discovery workflow runs on a **fixed 4-hour interval**, and its
  interval is explicitly not overridable; a `DiscoveryPropertyOverride`
  attempt on it fails XSD validation on import. There's also no shipped
  "run now" task for this specific discovery. Once the Run As account is
  fixed, there's no way to force it faster; budget the wait.
- Granting elevated permissions (the `sysadmin` grant and AD permission
  changes above) is exactly the category of action that gets blocked by AI
  coding agent safety guardrails if you're using one to help with a build
  like this, and that block can hold even after explicit chat approval and
  loosened local tool permissions, because the classifier for this category
  operates independently of both. If that's your situation, plan for a
  human to run this specific step by hand rather than fighting the
  automation.

---

## Step 11: Watch your internal web apps, not just servers

SCOM's built-in **Web Application Availability Monitoring** (WAAM), no
separate download, it's part of core SCOM, runs synthetic HTTP checks from
chosen agents ("watcher nodes") against URLs you specify. Useful for
internal web apps you can't otherwise get health data from.

**This one really is console-wizard-only** in this SCOM version; no
PowerShell authoring path exists for it.

1. **Authoring workspace → Management Pack Templates → Add Monitoring
   Wizard → Web Application Availability Monitoring.**
2. **General Properties**: name it, and explicitly point the **Management
   Pack** dropdown at a *new* unsealed MP; leaving it on the default can
   silently fail to save the template.
3. **Web Application → Transactions**: one per URL, an HTTP request (verb +
   URL) plus, optionally, a "response should contain" text match.
4. **Watcher Nodes**: shared by every transaction in this one Web
   Application object. For different watcher-node sets per URL group,
   create separate Web Application objects; you cannot mix watcher-node
   assignments within one.
5. On **View and Validate Tests**, the **Create** button being greyed out is
   expected; click **Next** to Summary, where **Create** is available.

**Before designing any check, find out what an anonymous request to the
target actually returns.** WAAM's default pass/fail is "did the target
respond within the timeout," and it does **not** fail on a non-2xx status
code by default. A 401 or a 302 still counts as "answered."

```powershell
try {
    $r = Invoke-WebRequest -Uri $url -UseBasicParsing -TimeoutSec 15 -ErrorAction Stop
} catch [System.Net.WebException] {
    $resp = $_.Exception.Response
    $body = (New-Object System.IO.StreamReader($resp.GetResponseStream())).ReadToEnd()
    # inspect $resp.StatusCode and $body before deciding how to build the check
}
```

Windows-auth-protected pages return a `401` with an empty body; there's
nothing to content-match against, so build those as reachability-only. Know
going in that reachability-only checks against a 401-challenge page can be
genuinely flaky (results flipping between success and failure across
watcher nodes and over time), because the underlying HTTP client sometimes
completes NTLM/Negotiate transparently and sometimes doesn't. If you need a
reliable check against a Windows-auth page, configure a Run As account with
real credentials for the WAAM template; bare reachability isn't a stable
signal on its own.

Also check reverse-proxied apps' bare root URL before assuming it's a good
check target: a proxy in front of a real backend can serve a generic
default placeholder page at the root instead of routing through, which
would report "healthy" via a root-URL check even with the real backend
completely down.

---

## Step 12: License SCOM itself

Apply your key:

```powershell
Set-SCOMLicense -ManagementServer <fqdn> -ProductId <key> -Credential (Get-Credential)
```

**This failed at first, and the fix is the one you already applied in Step
6.** Before that fix, `Get-SCOMLicense` returned a generic "file not
found" error and `Set-SCOMLicense` either threw a null-reference exception
or hung for several minutes, confirmed in a genuine interactive console
session, so it wasn't a remoting artifact. The cause was exactly Step 6's
gotcha: the calling account was a member of Operations Manager
Administrators only through a nested group, and licensing operations
specifically require *direct* membership. Once the account was added
directly, `Set-SCOMLicense` completed cleanly on the next attempt, no
other change needed.

If you skipped Step 6 assuming it was optional hardening advice, this is
the step where that assumption costs you actual time.

---

## Step 13 (optional): PIV/smartcard certificate authentication for the web console

Step 5 got Windows Integrated Auth working for domain-joined machines. This
step adds a second, independent sign-in path on top of it: client
certificate authentication, so anyone with a YubiKey (or any other PIV/smart
card) holding a certificate from your internal CA can authenticate to the
web console with the card instead of a Kerberos ticket. This build already
had a YubiKey-based smart card logon setup for domain sign-in, using a
`SmartcardLogon` certificate template; the same template and the same
issued certificates work here without any changes, because a template built
for domain smart card logon already carries the **Client Authentication**
EKU (`1.3.6.1.5.5.7.3.2`) alongside **Smart Card Logon**
(`1.3.6.1.4.1.311.20.2.2`) by default. If you're starting from scratch,
confirm your template has both before proceeding:

```powershell
Get-ADObject -Filter "Name -eq 'SmartcardLogon'" `
  -SearchBase "CN=Certificate Templates,CN=Public Key Services,CN=Services,CN=Configuration,$((Get-ADRootDSE).rootDomainNamingContext)" `
  -Properties pKIExtendedKeyUsage |
  Select-Object -ExpandProperty pKIExtendedKeyUsage
# Expect both 1.3.6.1.5.5.7.3.2 and 1.3.6.1.4.1.311.20.2.2 in the list
```

**This is additive, not a replacement**: the goal is "cards work, and
everyone who was signing in with Windows Auth keeps working exactly as
before," not "force everyone onto a card." Every command below adds a
capability; none of them touch the Windows Authentication config from
Step 5.

Each step below gives you both paths; pick whichever you're more
comfortable with, or mix and match (e.g. script the feature install, then
switch to the GUI for the settings you'll want to eyeball). They land on
identical configuration either way; nothing about doing one step via GUI
forces you into GUI for the next one.

### 1. Install the IIS certificate-authentication features

Despite the similar names, `Web-Cert-Auth` and `Web-Client-Auth` do
different things. `Web-Cert-Auth` ("IIS Client Certificate Mapping
Authentication") does manual, IIS-config-stored one-to-one/many-to-one
mappings. `Web-Client-Auth` ("Client Certificate Mapping Authentication",
yes, the shorter name is the *other* one) is the Active Directory–integrated
version, which automatically maps a presented certificate to an AD account
by matching the UPN in the certificate's Subject Alternative Name, no
manual mapping table to maintain, which matters a lot if you add users over
time. Install both.

**PowerShell:**
```powershell
Install-WindowsFeature -Name Web-Client-Auth, Web-Cert-Auth -IncludeManagementTools
```

**GUI:** Server Manager → **Add Roles and Features** → Server Roles → expand
**Web Server (IIS)** → **Web Server** → **Security** → check **Client
Certificate Mapping Authentication** and **IIS Client Certificate Mapping
Authentication** → Next through to **Install**. No reboot needed for either.

### 2. Enable Active Directory Client Certificate Authentication

This is the setting that turns on the UPN-based automatic mapping from step
1. It's server-wide, not per-site.

**PowerShell:**
```powershell
& "$env:windir\system32\inetsrv\appcmd.exe" set config `
  -section:system.webServer/security/authentication/clientCertificateMappingAuthentication `
  /enabled:"True" /commit:apphost
```

**GUI:** there's no dedicated icon for this one in IIS Manager; use the
**Configuration Editor** instead, which can reach any config section
including this one. Open IIS Manager, select the **server node** (top of
the tree, not a site), double-click **Configuration Editor**, and in the
**Section** dropdown navigate to
`system.webServer/security/authentication/clientCertificateMappingAuthentication`.
Set **enabled** to **True**, then click **Apply** in the Actions pane on the
right.

### 3. Unlock the `security/access` section

IIS locks this section (`overrideModeDefault="Deny"`) at the server level
by default. Skip this and go straight to step 4, and you'll get a "cannot be
used at this path... locked at a parent level" error.

**PowerShell:**
```powershell
& "$env:windir\system32\inetsrv\appcmd.exe" unlock config -section:system.webServer/security/access
```

**GUI:** same Configuration Editor from step 2, navigate the **Section**
dropdown to `system.webServer/security/access`. If it's locked, you'll see
an **Unlock Section** link in the Actions pane on the right (instead of
Apply); click it once. You only need to do this once server-wide, not per
site.

### 4. Set the app to negotiate client certificates, not require them

This is the one setting in this whole step where getting the value wrong
locks people out rather than just not working: requiring certificates
instead of negotiating them would mean *only* card holders could reach the
console at all, immediately breaking every Windows-Auth user from Step 5.

**PowerShell:**
```powershell
Set-WebConfigurationProperty -Filter 'system.webServer/security/access' `
  -PSPath 'IIS:\Sites\Default Web Site\OperationsManager' -Name sslFlags -Value 'Ssl,SslNegotiateCert'
```

**GUI:** this one has a proper dedicated icon, and is arguably the easier
path for this specific step. IIS Manager → expand to the
**OperationsManager** application under Default Web Site → double-click
**SSL Settings** → check **Require SSL** → under **Client certificates**,
select **Accept** (not **Require**; Accept is the GUI's name for
negotiate/optional; Require is exactly the mistake called out above) →
**Apply**.

### 5. Confirm Windows Authentication is still enabled

A quick regression check, not an optional nicety, given how easy it is for
one of the config commands above to touch more than intended.

**PowerShell:**
```powershell
Get-WebConfigurationProperty -Filter '/system.webServer/security/authentication/windowsAuthentication' `
  -PSPath 'IIS:\Sites\Default Web Site\OperationsManager' -Name enabled
# Value should still read True
```

**GUI:** IIS Manager → the same **OperationsManager** application →
double-click **Authentication** → confirm **Windows Authentication** still
shows **Enabled** in the Status column. (While you're here, this is also
where you'd notice if it had somehow been turned off: the icon shows every
auth method's state in one screen, arguably a faster check than the
PowerShell one-liner.)

### 6. Confirm the site itself is still healthy

Expect a clean `401` on an unauthenticated request, not a `500` or a hang.

**PowerShell:**
```powershell
try { Invoke-WebRequest -Uri 'https://opsmgr.myhomelab.hv.lab/OperationsManager/' -ErrorAction Stop }
catch { $_.Exception.Response.StatusCode }   # expect: Unauthorized
```

**GUI:** open the URL in a browser on any domain-joined machine. You should
land on a sign-in prompt (Windows Auth, a certificate picker, or both in
sequence) rather than an IIS error page. An IIS error page here means one
of steps 1–4 didn't land the way you expected; recheck them before moving
on, rather than troubleshooting the certificate side of things first.

**What you can't verify from PowerShell**: the actual "insert card, browser
prompts for a certificate, pick it, you're in" flow needs a human at a real
browser with a physical card reader. Browser certificate-picker dialogs are
native OS UI, the same category as a Windows credential prompt, and nothing
scriptable can drive or inspect them. If you're verifying this yourself:
open the console URL in a browser on a machine with the card reader
attached, and you should get a certificate selection prompt before (or
instead of) the usual Windows Auth handshake. If the prompt never appears,
double-check step 4's `sslFlags` value landed; a missing
`SslNegotiateCert` flag is the most common reason IIS never asks for a
certificate at all.

---

## Step 14: SSRS, SCOM Reporting, and Application Advisor

This step covers the reporting stack: configuring SQL Server Reporting
Services, installing SCOM's Reporting component on top of it, and getting
the web console's **Application Diagnostics** and **Application Advisor**
(.NET app monitoring) pages working. It assumes the SSRS *feature* is
already installed (a standard SQL Server Reporting Services setup run,
not covered here) and the management server from Step 1 already exists.

**Read this before doing anything else in this step.** Three separate,
unrelated failure classes stack on top of each other in this exact order,
and each produces a generic, misleading error that points troubleshooting
in the wrong direction if you don't recognize the pattern:

1. **A DPAPI cross-account encryption mismatch** when configuring SSRS
   headlessly as the wrong account → SSRS silently can't decrypt its own
   config, and logs a vague "No DSN present" loop forever.
2. **Windows' NTLM loopback protection** blocks the SCOM Reporting
   installer's own pre-flight HTTP check against the management console's
   FQDN → the installer aborts with a plain `401 Unauthorized`, nothing
   hinting it's a loopback issue.
3. **Two independent SQL Server permission surfaces**: the `ReportServer`
   catalog database and the `OperationsManagerDW` data warehouse database,
   that look like "the same reporting permissions" but are not. An account
   can be fully working against one and completely denied on the other,
   and the resulting error (`rsProcessingAborted`) doesn't say which
   database or which account.

### Why bother: what these three pieces actually give you

Easy to treat this as a checkbox exercise, so worth being explicit about
what it's actually for before spending an afternoon on DPAPI and NTLM
loopback. **SSRS / SCOM Reporting** turns the Data Warehouse's raw history
into the documents an ops team actually has to produce on a schedule:
monthly availability/uptime reports for a change advisory board or an SLA,
capacity trend reports (CPU/memory/disk over a rolling 90 days) that
justify a hardware refresh before it becomes an incident, and scheduled
subscriptions that land a PDF in an inbox every Monday without anyone
opening the console. **Application Advisor** gives a first-pass answer to
"why is this .NET app slow" without a dedicated APM agent inside the
app. Resource Utilization Analysis shows whether it's host CPU/memory
pressure versus the app itself, Problem Analysis Reports group exceptions
by frequency and correlate a spike against a deployment window, and Client
Side Monitoring turns "the app feels slow" into an actual measured
page-load number. **Application Diagnostics** is the event-search
counterpart: one search box across the whole fleet's .NET application
events, instead of RDPing into individual IIS servers during an incident.

### Provision the account the clean way, from the start

You need two accounts doing genuinely different jobs. Provision both
*before* touching SSRS configuration, and grant the second one rights in
**both** databases at creation time. Reusing an existing "reader" account
ad hoc, then discovering the `OperationsManagerDW` gap only after a report
fails to render, is exactly the mistake this section exists to help you
skip.

| Account | Type | Used for |
|---|---|---|
| e.g. `svc-scomdw$` | gMSA | SCOM Reporting's **Data Reader** account: what report data source connections authenticate as at runtime. Usually already exists as the Data Warehouse RunAs account. |
| A second, regular (non-gMSA) account, e.g. `svc-scomrpt-config` | Standard user, real password | Running SSRS config steps that touch encrypted secrets, and the SSRS **Unattended Execution Account**. Must have a retrievable password; a gMSA can't fill either role. |

**PowerShell (AD account):**
```powershell
New-ADUser -Name "svc-scomrpt-config" -SamAccountName "svc-scomrpt-config" `
    -AccountPassword (Read-Host -AsSecureString "Password") -Enabled $true -PasswordNeverExpires $true
```
**GUI:** Active Directory Users and Computers (`dsa.msc`) → right-click
your service-accounts OU → **New → User** → set the name, password, and
check **Password never expires** → **Finish**.

**SQL (DB grants):**
```sql
USE [ReportServer];
CREATE USER [DOMAIN\svc-scomrpt-config] FOR LOGIN [DOMAIN\svc-scomrpt-config];

USE [OperationsManagerDW];
CREATE USER [DOMAIN\svc-scomrpt-config] FOR LOGIN [DOMAIN\svc-scomrpt-config];
ALTER ROLE OpsMgrReader ADD MEMBER [DOMAIN\svc-scomrpt-config];
```
**GUI:** SQL Server Management Studio → Object Explorer → **Databases →
OperationsManagerDW → Security → Users** → right-click → **New User...** →
set **User name** / **Login name** to the account → OK → right-click the
new user → **Properties → Membership** → check **OpsMgrReader** → OK.

**The `OperationsManagerDW` grant is the one that's easy to skip and the
one this whole step exists to make sure you don't.** `OpsMgrReader` alone
is sufficient for report rendering; don't over-grant to `db_owner`.

Confirm the SSRS instance is local to this management server: SCOM
Reporting setup **requires** this, a remote instance won't pass setup's
validation:

```powershell
Get-CimInstance -Namespace "root\Microsoft\SqlServer\ReportServer\RS_SSRS\V16\Admin" `
    -ClassName MSReportServer_ConfigurationSetting | Select-Object InstanceName, SiteName
```

### The core gotcha: run every SSRS config step as a genuine logon of the target account

**Symptom if you get this wrong:** `rsreportserver.config`'s `<Dsn>` (or
`<UnattendedExecutionAccount>`) tag looks populated, a base64 blob
starting `AQAAANCMnd8B...`, the DPAPI signature, but the SSRS service logs
"No DSN present in configuration file" on every startup, repeating
forever. `IsInitialized` stays `False`.

**Why:** SSRS's WMI configuration provider
(`MSReportServer_ConfigurationSetting`) encrypts secrets using DPAPI
scoped to **whichever account is actually calling the WMI method**, not
the SSRS service account, and not whatever desktop user you're logged in
as. Drive this from a plain PowerShell call at an `administrator` desktop
session, and the encryption is scoped to `administrator`'s DPAPI context.
The SSRS service, running as its own account, can never decrypt it later.

**The fix:** get a genuine new logon session for the target account and
call the WMI method from inside it:

```powershell
# $cred = a PSCredential for the account you want the config scoped to
Invoke-Command -ComputerName localhost -Credential $cred -ScriptBlock {
    whoami   # confirms which account is actually running this
    $cfg = Get-CimInstance -Namespace "root\Microsoft\SqlServer\ReportServer\RS_SSRS\V16\Admin" `
        -ClassName MSReportServer_ConfigurationSetting
    # ... call whichever SetXxx method you need here, using $cfg ...
}
```

`Invoke-Command -ComputerName localhost -Credential $cred` produces a real
LSA logon for that account, enough for the correct DPAPI context,
**without an RDP/console session.** Apply this to every step below that
touches an encrypted value.

### Configure SSRS: database, URLs, and the Unattended Execution Account

**Set the database connection**: the one genuinely GUI-driven step.
`RSConfigTool.exe` (under `...\Microsoft SQL Server Reporting
Services\SSRS\Shared Tools\`), run **inside a real logon session of the
target account**: Database page → Change Database → Create a new report
server database → target your SQL instance/AG listener → **use explicit
Windows credentials**, not "Service Account" (`CredentialsType=0` fails
reliably in this kind of environment; explicit username/password,
`CredentialsType=2`, succeeds). Verify `IsInitialized` reads `True` before
continuing.

**Set the Web Service and Web Portal URLs.**

**GUI:** same `RSConfigTool.exe` session: **Web Service URL** page →
confirm the virtual directory and TCP port → **Apply**; then **Web Portal
URL** page → confirm its virtual directory (don't assume it mirrors the
web service one, read what the tool proposes) → **Apply**. The tool
restarts the service for you.

**PowerShell**, via WMI, from inside the credentialed session, same
methods, different `Application` value (`"ReportServerWebService"` for
`/ReportServer`, `"ReportServerWebApp"` for the portal):

```powershell
$cfg = Get-CimInstance -Namespace "root\Microsoft\SqlServer\ReportServer\RS_SSRS\V16\Admin" `
    -ClassName MSReportServer_ConfigurationSetting
$lcid = (Get-Culture).LCID

Invoke-CimMethod -InputObject $cfg -MethodName SetVirtualDirectory -Arguments @{
    Application = "ReportServerWebApp"; VirtualDirectory = "Reports_SSRS"; Lcid = $lcid
}
Invoke-CimMethod -InputObject $cfg -MethodName ReserveUrl -Arguments @{
    Application = "ReportServerWebApp"; UrlString = "https://opsmgr.myhomelab.hv.lab:443"; Lcid = $lcid
}

# Restart to pick up a newly-set virtual directory - expect a transient 503 for ~10-20s
Restart-Service SQLServerReportingServices -Force
```

That lands the two SSRS endpoints at
`https://opsmgr.myhomelab.hv.lab/ReportServer/` (the web service, used by
the URL-access API check later in this step) and
`https://opsmgr.myhomelab.hv.lab/Reports_SSRS/` (the actual report portal
a person opens in a browser).

Then verify both endpoints respond `200` with `-UseDefaultCredentials`. A
`401` here specifically on a *short hostname* (not FQDN) is a real,
different auth problem; don't confuse it with the loopback issue below,
which is FQDN-specific.

**Set the Unattended Execution Account**: separate from the Data Reader
account, easy to skip since nothing in the base install requires it, but
Application Advisor reports need it.

**GUI:** same `RSConfigTool.exe` session: **Execution Account** page →
check **Specify an execution account** → enter the domain username and
password → **Apply**. The simplest path for this one setting, since the
tool already has you in the right logon context.

**PowerShell:**
```powershell
Invoke-Command -ComputerName localhost -Credential $execAccountCred -ScriptBlock {
    param($pw)
    $cfg = Get-CimInstance -Namespace "root\Microsoft\SqlServer\ReportServer\RS_SSRS\V16\Admin" `
        -ClassName MSReportServer_ConfigurationSetting
    Invoke-CimMethod -InputObject $cfg -MethodName SetUnattendedExecutionAccount -Arguments @{
        UserName = "DOMAIN\svc-scomrpt-config"; Password = $pw
    }
} -ArgumentList $plaintextPassword

Restart-Service SQLServerReportingServices -Force
```

**Don't skip verifying this specifically.** A broken Unattended Execution
Account still returns HTTP `200` on every Application Advisor page, so a
green URL check alone doesn't prove it's configured correctly.

### Fix the Windows NTLM loopback block before running SCOM Reporting setup

**Symptom:** the silent installer aborts almost immediately:

```
Error: CheckHttpAddressResponse failed: ... (401) Unauthorized.
OM component OMREPORTING is not valid to install on this box.
```

...even though SSRS itself is fully healthy. This is the same class of
NTLM loopback/back-connection protection (KB896861) behind Step 5's fix
#3, but a **different registry value**: this one lives under
`HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\MSV1_0`, not the plain `Lsa`
key from Step 5. Setting one does not set the other. The management
server's real computer name (`OPSMGR01`) is automatically exempt from
loopback protection, but the Reporting installer validates against a
*different* FQDN alias (the console name from Step 1, e.g.
`opsmgr.myhomelab.hv.lab`). Windows silently refuses to send NTLM
credentials to that alias from a same-box process.

Confirm before fixing: compare identical requests, varying only the
hostname (short name should succeed, FQDN alias should 401):

```powershell
$req2 = [System.Net.HttpWebRequest]::Create("https://opsmgr.myhomelab.hv.lab:443/ReportServer")
$req2.Credentials = [System.Net.CredentialCache]::DefaultCredentials
try { ($req2.GetResponse()).StatusCode } catch { $_.Exception.Message }   # expect 401
```

The fix, PowerShell/registry only, there's no GUI path for this one
(`regedit.exe` to the same key by hand is the closest thing, and it's just
a manual version of the same write):

```powershell
$lsaPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\MSV1_0"
$existing = (Get-ItemProperty -Path $lsaPath -Name BackConnectionHostNames -ErrorAction SilentlyContinue).BackConnectionHostNames
$hostnames = @($existing) + @("opsmgr.myhomelab.hv.lab") | Select-Object -Unique
New-ItemProperty -Path $lsaPath -Name BackConnectionHostNames -Value $hostnames -PropertyType MultiString -Force
```

Takes effect immediately, no reboot needed.

### Install the SCOM Reporting component, then grant the account rights in both databases

**GUI:** run `Setup.exe` without any switches → **Install** → the wizard
detects the existing management server and shows **Reporting server** as
an available checkbox alongside the already-installed components → check
it → **Next** through the same Management Server, SSRS instance, and Data
Reader account prompts the CLI switches below ask for → **Install**. Same
wizard as Step 1, just re-run to add one more component.

**PowerShell / silent install:**
```powershell
C:\<path-to-media>\Setup.exe /silent /install /components:OMReporting `
  /ManagementServer:<mgmt-server-fqdn> /SRSInstance:<hostname>\<instance-name> `
  /DataReaderUser:<DOMAIN>\<data-reader-gmsa>$ /AcceptEndUserLicenseAgreement:1 `
  /SendCEIPReports:1 /SendODRReports:0 /UseMicrosoftUpdate:0 /EnableErrorReporting:Always
```

**`/DataReaderUser` is the correct switch name.** A similar-looking
`/DataReaderAccountUser` is silently ignored (no error, it just never
registers as present in the log), a classic copy-paste trap from older
SCOM documentation. Expect ~10-15 minutes end to end: the MSI is fast, but
the post-install step republishes the entire built-in report catalog,
hundreds of log lines, one every few seconds, don't kill it early.

If you skipped the up-front grant, opening any report now fails with a
generic `Cannot initialize report (rsProcessingAborted)`. The real reason
lives in
`...\SSRS\LogFiles\ReportingServicesService_<timestamp>.log`, not any SCOM
log; look for a login-failure line naming `OperationsManagerDW`, then
apply the `CREATE USER`/`ALTER ROLE` grant from above.

### Application Diagnostics and Application Advisor: where NOT to look for the fix

`AppDiagnostics` and `AppAdvisor` are IIS virtual applications under
`Default Web Site`, part of the Step 1 Web Console install, running under
IIS application pool `OperationsManagerAppMonitoring`. **Neither app's
`web.config` has any hardcoded Reporting Server URL or credentials**:
they resolve the SSRS instance dynamically. There is no application-level
config file to edit for this integration: every fix in this step is
either an SSRS-side setting or a SQL Server permission, never a
`web.config` change.

### Verify

- `https://opsmgr.myhomelab.hv.lab/AppDiagnostics/Pages/Search/AllEvents.aspx`
  and `https://opsmgr.myhomelab.hv.lab/AppAdvisor/` both load, `200`, real page content,
  **not** redirected to `errorpage.htm`. If you land on
  `errorpage.htm?aspxerrorpath=...`, check the Application event log
  (source `ASP.NET 4.0.30319.0`, event `1309`) for the underlying
  `SoapException`, almost always the Unattended Execution Account or the
  database permission gap. **A `200` here does not mean the page actually
  works.** This failure mode redirects client-side, so check content, not
  just status codes.
- Desktop console → Reporting workspace → `Application Monitoring` →
  `.Net Monitoring` → `Application Advisor Reports` → three populated
  subfolders, and any individual report renders on double-click.
- The most reliable check, bypassing every GUI layer, render a report
  directly against the SSRS URL-access API:
  ```powershell
  Invoke-WebRequest -UseDefaultCredentials -UseBasicParsing -TimeoutSec 30 -Uri (
    "https://opsmgr.myhomelab.hv.lab/ReportServer?" +
    "%2fApplication+Monitoring%2f.Net+Monitoring%2fApplication+Advisor+Reports" +
    "%2fResource+Utilization+Analysis%2fApplication+CPU+Utilization+Analysis" +
    "&rs:Format=HTML4.0&rs:Command=Render"
  )
  ```
  `200` with real report-viewer HTML and no `rsProcessingAborted`/`Login
  failed` text in the body = fully working.

### The gotcha this build actually hit: the catalog database wasn't in the AG at all

Getting a `200` on the portal shell (`/Reports_SSRS/`) is not the same as
reporting actually working. This build's `/ReportServer/` endpoint, the
one the render check above depends on, hung and then failed with:

```
ReportServerDatabaseLogonFailedException: The report server cannot open a
connection to the report server database. The log on failed.
Login failed for user 'MYHOMELAB\svc-scomdw$'.
Cannot open database "ReportServer" requested by the login.
```

**Confirm which of two things you're looking at before assuming a
permissions fix will be sufficient:**

```powershell
$cfg = Get-CimInstance -Namespace "root\Microsoft\SqlServer\ReportServer\RS_SSRS\V16\Admin" `
    -ClassName MSReportServer_ConfigurationSetting
$cfg.DatabaseServerName
```

Two separate things were wrong here, and fixing only the first isn't
enough. `svc-scomdw$` (the Data Reader gMSA from earlier in this step)
was missing its `RSExecRole` membership in the `ReportServer` catalog
database itself, not just `OperationsManagerDW`, granted the same way as
the SQL grants earlier in this step. But the one that actually mattered:
SSRS was configured to reach its catalog via the SQL AG listener, while
`ReportServer` and `ReportServerTempDB` had never been added to the
Availability Group at all; they existed only on one specific node, a
leftover from an earlier, abandoned SSRS install on a different instance
that got cleaned up without anyone doing the separate follow-up of
joining the *new* install's databases to the group. The listener happily
routes to whichever node is currently primary, and the moment that isn't
the node holding these databases, the exact same login-failure error
comes back, permissions or not. This is the same failure class covered
in the Azure DevOps Server post in this series (a database that exists
on one AG node, reached through a mechanism that can route to a
different one), applied here to SSRS's own plumbing instead of a
monitored application database.

**Don't reach for the WMI method first.** The obvious tool for
repointing the connection is
`MSReportServer_ConfigurationSetting.SetDatabaseConnection`, a
documented WMI method built for exactly this. It failed identically
(`HRESULT 0x8004022C`) across every identity, logon type (direct call,
PS remoting, `Start-Process -Credential`, a scheduled task with a real
password), and parameter ordering tried, with nothing in the trace log
suggesting any of those calls reached real connection-validation logic.
That kind of uniform failure across every variable that normally matters
is itself the signal: the API path isn't the right lever here.
**Reporting Services Configuration Manager, run interactively as a real
account with a real password, is the tool that's actually proven to
work for this**, not the WMI provider underneath it. That interactivity
requirement is literal: this needed a human physically at the console
clicking through the wizard, not another script driving it.

Two more real snags surfaced doing it that way, worth knowing before you
hit them cold:

- **The wizard's Apply button needs more than `RSExecRole`.** Runtime
  data access and schema/permission changes are different permission
  surfaces. Pointing the wizard at the database and clicking Apply
  produced `Error Number: 15247, User does not have permission to
  perform this action`, even with `RSExecRole` already granted.
  `db_owner` on the database still wasn't enough; the wizard needs to
  manage the service account's own SQL login, a server-level object,
  which needs `sysadmin` at the instance level to touch. That grant is
  scoped to whichever node you ran the wizard against, not carried by
  AG membership, backup/restore, or automatic seeding, so expect to
  regrant it on every node that can become primary, and to create the
  server login at all on any node that's never had this account touch
  it before.
- **A shared account can be a hidden second point of failure.**
  Resetting this account's password to get a genuine interactive logon
  broke a second thing silently: the same account was also configured,
  months earlier, as SSRS's Unattended Execution Account, the identity
  used to resolve report parameters and subreports with no interactive
  user present. A `200` on the report endpoint with a `Login failed for
  Unattended Execution account` error rendered inside the page is a
  disguised failure, not a working one, check page content, not just
  status code, exactly the same lesson as the AppDiagnostics/AppAdvisor
  verification earlier in this step.

> **⚠ Joining a database to a production Availability Group is not a
> low-risk step**, even though the commands themselves are short. Verify
> the AG is healthy and fully synchronized before you start, touch one
> node at a time, and re-verify health after each step rather than
> assuming success from a clean exit code. This build hit real bugs
> doing exactly this (a silently-failing `Invoke-Sqlcmd` parameter that
> printed false success messages, a cross-machine restore permission
> trap, AG DDL needing `master` context) precisely because those checks
> weren't automatic.

**Once the catalog databases are actually AG members** (same
backup/`ADD DATABASE`/verify-sync procedure as the Azure DevOps Server
post's Step 8, minus `ReportServerTempDB`, deliberately left out since
it's transient, rebuildable data that standard guidance says not to
AG-protect, though it still needs to physically exist as a standalone
database on every node that can become primary, "not AG-protected"
isn't the same claim as "doesn't need to exist everywhere"), verify the
whole chain, not just that the sync state is green:

```powershell
Get-CimInstance -Namespace "root\Microsoft\SqlServer\ReportServer\RS_SSRS\V16\Admin" `
    -ClassName MSReportServer_ConfigurationSetting |
    Select-Object DatabaseServerName, IsInitialized
```

`DatabaseServerName` reading the listener name and `IsInitialized: True`
means the connection itself is fixed. Confirm the actual page loads too:

```powershell
Invoke-WebRequest -Uri 'https://opsmgr.myhomelab.hv.lab/ReportServer/' -UseDefaultCredentials -UseBasicParsing
# expect: 200
```

This build's `ReportServer` database is confirmed `SYNCHRONIZED` across
all three AG replicas, and the endpoint above returns a clean `200`,
durable against a future failover now, not just working by accident on
whichever node happened to be primary when it was fixed.

### A second, unrelated gotcha: fixing SSRS didn't fix Application Advisor or Diagnostics

Application Advisor and Application Diagnostics aren't part of SSRS,
despite living behind the same web console and getting fixed in the
same sitting on this build. They're separate IIS components (Avicode
SEManager) talking directly to `OperationsManager` and
`OperationsManagerDW` over their own OLE DB connection, authenticated
as the local machine's own computer account, not the Data Reader gMSA
SSRS uses. Getting SSRS working doesn't get these two working; they
have their own, independent failure path.

**Wall one, the same AG-login gap in different clothes**: the
Application event log named the exact problem:

```
Exception type: OleDbCommandException
Login failed for user 'MYHOMELAB\OPSMGR01$'.
Connection: Provider=MSOLEDBSQL; Server=SQLAGL01; database=OperationsManagerDW;
```

`OPSMGR01$`'s database-level users (`apm_datareader`,
`apm_datawriter`) were correct in both databases, present via AG
seeding, but the server-level *login* for that machine account only
existed on the node it was originally created against, not the other
two. This is the same shape of bug as SSRS's own catalog databases
above: a server-level principal doesn't travel with AG membership the
way a database-level one does. Confirm before assuming a fresh grant is
needed everywhere:

```powershell
Invoke-Sqlcmd -ServerInstance 'SQL01' -Query "SELECT name FROM sys.server_principals WHERE name = 'MYHOMELAB\OPSMGR01`$'"
# repeat against SQL02 and SQL03 individually, not through the listener
```

Where it's missing, the fix is one statement, no permission remapping
needed since the existing database users bind to the new login
automatically once it exists:

```sql
CREATE LOGIN [MYHOMELAB\OPSMGR01$] FROM WINDOWS;
```

Application Advisor came back clean immediately after. Application
Diagnostics didn't, a different exception took its place, which is
progress, not a regression: a new failure at the same URL means the
request got further than it ever had before.

**Wall two, an access-control list nobody ever populated**: the new
exception wasn't a database error at all:

```
Exception type: ApplicationException
Exception message: Security Check Failed
   at Avicode.Intercept.SEManager.WebViewer.SemBase.SemPage.CheckSecurity()

User: MYHOMELAB\Administrator     Is authenticated: True     Authentication Type: Forms
```

A fully-authenticated domain administrator, rejected, not by Windows or
SQL, but by SCOM's own internal authorization check for this specific
feature. The role that gates it, `OperationsManagerApmOperator`
("Application Monitoring Operator" in the console), had zero members,
not misconfigured, just never populated since original setup. Every
request from every account was always going to fail this check, since
there was no one on the allow list to belong to. Confirm and fix:

```powershell
$role = Get-SCOMUserRole -Name 'OperationsManagerApmOperator'
$role.Users   # empty means this is your problem
Set-SCOMUserRole -UserRole $role -User @("MYHOMELAB\SCOMAdmins")
```

Add the lab's existing admin group rather than an individual account,
so anyone who should have access already does. **This app runs its own
Forms-authentication session**, separate from the Windows Integrated
Auth fixed back in Step 5, so an already-open browser tab can keep
showing the old failure after the fix lands; a fresh login clears it.

This build confirmed both walls cleared live: `OPSMGR01$` has a server
login on all three SQL nodes, `OperationsManagerApmOperator` now has
`SCOMAdmins` as a member, and both
`https://opsmgr.myhomelab.hv.lab/AppDiagnostics/Pages/Search/AllEvents.aspx`
and `https://opsmgr.myhomelab.hv.lab/AppAdvisor/` return a clean `200`
with real page content. Worth a one-time sweep of this management
group's other user roles for the same zero-members gap while you're in
here; nothing surfaces it until someone actually exercises the gated
feature for the first time.

![Application Advisor's full report catalog, working end to end]({{ '/assets/img/gallery/scom-application-advisor-reports.png' | relative_url }})
_Client Side Monitoring, Problem Analysis Reports, and Resource Utilization Analysis, all populated and rendering, not just a 200 on an empty page_

![Application Diagnostics loaded clean, past the security check]({{ '/assets/img/gallery/scom-application-diagnostics-working.png' | relative_url }})
_The event search UI itself, signed in as MYHOMELAB\Administrator, no Security Check Failed error in sight_

---

## Part two: from one management server to three

Everything above stood up a single management server, `OPSMGR01`, and it
carried the whole environment on its own for a couple of weeks. The steps
below turn that into a real 3-node deployment: `OPSMGR02` and `OPSMGR03`
already joined the same management group the same way `OPSMGR01`'s own
install did (`setup.exe /components:OMServer` against the same operational
database, not repeated here since it's identical to Step 1 minus the
database-creation prompts). What follows is everything after that point.

**What you'll end up with**:

- All monitored agents spread across all 3 management servers as
  *primary*, each with the other 2 configured as *failover*, not all
  pinned to one server with the others sitting as cold spares.
- Confirmed, not assumed, that the alert notification channel survives any
  one management server going down.
- A load-balanced DNS name for the web console
  (`scom-console.myhomelab.hv.lab`) that keeps working if any one
  management server is down, without breaking client-certificate
  (smartcard/YubiKey) authentication to it.

If you're building this as a single-server deployment and stopping there,
everything above already stands on its own. If you're following along to
add real redundancy the way this build eventually did, keep going.

![All 3 management servers healthy, load spread across the group]({{ '/assets/img/gallery/scom-management-servers-healthy.png' | relative_url }})
_The actual end state this section builds toward: OPSMGR01, 02, and 03, all Healthy_

---

## Step 15: Distribute agent load across management servers

By default, every agent's primary management server is whichever one
registered it, usually all of them, if you built the group with one
server before adding the others. The other management servers get
auto-added to each agent's *failover* list the moment they join the
resource pool, but nothing rebalances *primary* assignment for you.

**Why this matters**: without rebalancing, your "3 management servers"
setup is really "1 management server doing all the work, plus 2 cold
spares that only help after a failure." Spreading primary assignment
means all 3 are actually load-bearing all the time, and losing any one
only affects a third of your agents (who then fail over automatically)
instead of all of them.

```powershell
Import-Module OperationsManager
New-SCOMManagementGroupConnection -ComputerName opsmgr01.myhomelab.hv.lab

$allMS = Get-SCOMManagementServer
$ms1 = $allMS | Where-Object {$_.DisplayName -eq "opsmgr01.myhomelab.hv.lab"}
$ms2 = $allMS | Where-Object {$_.DisplayName -eq "opsmgr02.myhomelab.hv.lab"}
$ms3 = $allMS | Where-Object {$_.DisplayName -eq "opsmgr03.myhomelab.hv.lab"}

# Split your agent list into 3 roughly-even groups. Spread related
# pairs (e.g. two domain controllers, or nodes of the same cluster)
# across DIFFERENT primaries, otherwise losing one management server
# can coincide with losing monitoring for a whole functional tier at once.
$groupMS2 = @('Agent04','Agent05','Agent06')   # example names
$groupMS3 = @('Agent07','Agent08','Agent09')
# Agents not listed stay on their current primary (MS1 in this example)
```

### The gotcha: you can't just set Primary, then Failover

If you try the obvious approach:

```powershell
Set-SCOMParentManagementServer -Agent $agents -PrimaryServer $ms2
Set-SCOMParentManagementServer -Agent $agents -FailoverServer @($ms1,$ms3)
```

The **first** call fails for any agent that's actually changing
management server, with an error like:

```
The failover server <guid> cannot be the same as the primary server.
```

**Why this happens**: the moment a new management server joins the
resource pool, SCOM automatically adds it to every existing agent's
*failover* list. So before you've changed anything, an agent primaried to
MS1 already has `Failover: [MS2, MS3]`. When you then try to set
`Primary: MS2`, the cmdlet checks against the agent's *current* state and
rejects it: MS2 is still sitting in the failover list you haven't
cleared yet. Doing it in the "obvious" order (primary first) or the
"obvious" reversed order (failover first) both fail, because whichever
one you do first, the other role is still occupied by the value you're
about to move there.

**The fix**: a 3-step sequence per group of agents moving to a new
primary `$msX`, with target failover list `$others`:

```powershell
function Move-AgentPrimary($agents, $newPrimary, $dropFirst, $finalFailover) {
    # 1. Drop the incoming primary from the CURRENT failover list first,
    #    leaves one placeholder server so the list is never empty
    Set-SCOMParentManagementServer -Agent $agents -FailoverServer @($dropFirst)

    # 2. Now safe to set the new primary, it's no longer in the failover list
    Set-SCOMParentManagementServer -Agent $agents -PrimaryServer $newPrimary

    # 3. Re-fetch fresh agent objects (their in-memory state is stale
    #    after step 2) and set the REAL final failover list
    $fresh = Get-SCOMAgent | Where-Object { $agents.DisplayName -contains $_.DisplayName }
    Set-SCOMParentManagementServer -Agent $fresh -FailoverServer $finalFailover
}

$ms2Agents = Get-SCOMAgent | Where-Object { $groupMS2 -contains ($_.DisplayName -split '\.')[0] }
Move-AgentPrimary -agents $ms2Agents -newPrimary $ms2 -dropFirst $ms3 -finalFailover @($ms1,$ms3)

$ms3Agents = Get-SCOMAgent | Where-Object { $groupMS3 -contains ($_.DisplayName -split '\.')[0] }
Move-AgentPrimary -agents $ms3Agents -newPrimary $ms3 -dropFirst $ms2 -finalFailover @($ms1,$ms2)
```

Agents you're leaving on their current primary need no change at all;
their failover list is already correct from SCOM's auto-assignment.

### Verify

```powershell
Get-SCOMAgent | Select-Object DisplayName, PrimaryManagementServerName,
    @{N='FailoverServers';E={($_.GetFailoverManagementServers() | Select-Object -ExpandProperty Name) -join ', '}} |
    Sort-Object PrimaryManagementServerName
```

Confirm a roughly even split, and that every agent's failover list holds
the two management servers it *isn't* primaried to.

**If reassigned agents live on servers you consider "guarded"** (domain
controllers, clustered SQL nodes, anything with its own change-control
process), reassigning the management server doesn't restart or disrupt
`HealthService` on the agent, but verify your own guarded-system health
checks (cluster quorum, replication, whatever applies) before and after
anyway. Cheap insurance.

## Step 16: Verify notification resilience (don't assume it)

SCOM's built-in **Notifications Resource Pool** is dynamic: every new
management server auto-joins it, same as the general resource pool. In
principle, your existing alert notification channel/subscription
automatically becomes resilient to a management server going down. In
practice, **verify this, don't assume it**: pool membership can get
scoped away by a prior admin action, and there's no obvious symptom until
the one remaining node in the pool also goes down.

```powershell
Import-Module OperationsManager

# Map HealthService class-instance GUIDs to names (pool Members are NOT
# ManagementServer objects, a common trap if you try Get-SCOMManagementServer here)
$hsMap = @{}
Get-SCOMClassInstance -Class (Get-SCOMClass -Name 'Microsoft.SystemCenter.HealthService') |
    ForEach-Object { $hsMap[$_.Id.ToString()] = @{Name=$_.DisplayName; Health=$_.HealthState} }

$pool = Get-SCOMResourcePool -DisplayName "Notifications Resource Pool"
$pool.Members | ForEach-Object {
    $info = $hsMap[$_.Name.ToString()]
    if ($info) { "$($info.Name)  [HealthState: $($info.Health)]" }
}
```

You should see all of your management servers listed, all
`HealthState: Success`. Then confirm the actual channel/subscription is
still enabled and pointed where you expect:

```powershell
Get-SCOMNotificationChannel | Select-Object Name, DeliveryProtocol, Endpoint
Get-SCOMNotificationSubscription | Select-Object Name, Enabled
```

## Step 17: Make sure the management servers monitor each other

If you built your monitoring groups (functional groups, dashboards, etc.)
before adding the 2nd and 3rd management servers, check whether your
group-membership rules were scoped by hostname regex; if so, they
probably only match the original server, meaning your new management
servers aren't being monitored by their own management group. A
monitoring platform that can't see its own second and third nodes isn't
actually resilient, just larger.

If your group's discovery is a hand-authored, **unsealed** management
pack (common if you built it with a `GroupPopulator` discovery and a
regex `Pattern` element), this is a normal MP edit:

```powershell
Get-SCOMManagementPack -Name "YourGroups.MP" | Export-SCOMManagementPack -Path "C:\Temp\MPExport"
# Edit the exported XML:
#   <Pattern>^(OtherHost|MS1)\..*</Pattern>  ->  <Pattern>^(OtherHost|MS[1-3])\..*</Pattern>
#   bump <Version>, e.g. 1.0.0.5 -> 1.0.0.6
Import-SCOMManagementPack -Fullname "C:\Temp\MPExport\YourGroups.MP.xml"
```

**Don't expect instant results.** `GroupPopulator` discoveries run on
their own interval (commonly several hours if you haven't overridden it).
The import itself succeeds immediately and the fix is live; group
*membership* just catches up on the next discovery cycle. A
`Disable-SCOMDiscovery` / `Enable-SCOMDiscovery` toggle does not force an
early run; don't waste time trying to rush it.

## Step 18: Web console HA behind a reverse proxy

This is the part with real gotchas, so it gets the most detail.

### Why not Windows NLB

Windows NLB in unicast mode, the straightforward, no-extra-hardware
option, replaces the bound NIC's MAC address with a shared cluster MAC.
On a single-NIC VM, that can break the node's own normal network traffic
(DNS, SQL, inter-management-server communication), not just the
load-balanced VIP. The documented fix is a second, NLB-dedicated NIC per
node, but even with that in place, adding nodes to the cluster requires
remote WMI access between them, which isn't open by default on a fresh
Windows Server firewall, and even with every WMI-related firewall rule
enabled, we couldn't get past "RPC server is unavailable." **This is
solvable**, but by the time you've added a dedicated NIC to every node
and are debugging cross-machine RPC/DCOM permissions, you've built
meaningfully more moving parts than the problem needed.

### Why not an HTTP-layer proxy (like IIS ARR)

The real reason to reach for something else: **PKI/smartcard
authentication requires the actual TLS handshake, including client
certificate presentation, to happen directly against the backend IIS
server.** Any proxy that terminates TLS (which most HTTP-layer proxies
do, since they need to read the request to route/rewrite it) breaks that;
it would have to inspect the client cert itself and forward identity
via a header, which IIS's native AD Client Certificate Mapping doesn't
consume, and which shifts your trust boundary in a way that deserves
more scrutiny than a load-balancer config change should require.

### The answer: a TCP/stream-mode passthrough proxy

HAProxy (or nginx's `stream` module) in **TCP mode** reads only the
hostname from the unencrypted SNI field of the TLS ClientHello, routing
decisions happen before encryption starts, then relays the raw
encrypted bytes to a backend unchanged. The full TLS handshake, including
client cert negotiation, happens directly between the browser and the
real IIS server. The proxy is completely transparent to the security
boundary; it doesn't decrypt, doesn't need the certificate or private
key, and doesn't participate in the auth decision at all.

We're using pfSense's HAProxy package here since it's likely already
your network's edge point; the same TCP-mode principle applies if you'd
rather run this on nginx/HAProxy on a dedicated VM instead.

### 18a. Issue a shared certificate

Every backend node needs to present the **same** certificate for the
load-balanced hostname, plus (optionally) its own individual name, so
the same cert works whether someone hits the VIP or a node directly.

```powershell
# On one node, e.g. MS2:
$inf = @'
[Version]
Signature="$Windows NT$"
[NewRequest]
Subject = "CN=scom-console.myhomelab.hv.lab"
KeySpec = 1
KeyLength = 2048
Exportable = TRUE
MachineKeySet = TRUE
ProviderName = "Microsoft RSA SChannel Cryptographic Provider"
ProviderType = 12
RequestType = PKCS10
KeyUsage = 0xa0
[EnhancedKeyUsageExtension]
OID=1.3.6.1.5.5.7.3.1
[Extensions]
2.5.29.17 = "{text}"
_continue_ = "dns=scom-console.myhomelab.hv.lab&"
_continue_ = "dns=opsmgr01.myhomelab.hv.lab&"
_continue_ = "dns=opsmgr02.myhomelab.hv.lab&"
_continue_ = "dns=opsmgr03.myhomelab.hv.lab&"
[RequestAttributes]
CertificateTemplate = WebServerManualSAN
'@
$inf | Out-File cert-request.inf -Encoding ascii
certreq -new cert-request.inf cert-request.req
certreq -submit -config 'YourCA.domain.tld\Your-CA-Name' cert-request.req cert.cer
certreq -accept cert.cer

# Export it (with private key) to move to the other nodes:
$cert = Get-ChildItem Cert:\LocalMachine\My | Where-Object Subject -like '*scom-console.myhomelab.hv.lab*'
$pw = ConvertTo-SecureString -String 'temp-transfer-password' -Force -AsPlainText
Export-PfxCertificate -Cert $cert -FilePath cert.pfx -Password $pw
```

Move `cert.pfx` to the other nodes (PS remoting `Copy-Item -ToSession`
works well for this) and import it identically:

```powershell
Import-PfxCertificate -FilePath cert.pfx -CertStoreLocation Cert:\LocalMachine\My -Password $pw
```

**Why the same cert on every node**: the proxy is blind to which
specific backend it picks for a given connection (that's the point of
load balancing). If each node had its own individually-issued
certificate, the client would see a different cert depending on which
backend happened to answer, harmless for encryption, but confusing, and
it breaks a cert-pinning or SAN-validation check if you ever add one.

### 18b. Configure IIS bindings on every node

Each node needs three HTTPS bindings on port 443: a default (non-SNI)
catch-all, one for its own name, and one for the load-balanced name,
all using the shared certificate:

```powershell
Import-Module WebAdministration
$thumb = '7EE4F90AA82F3144B37338E19CF0DDF1094B1892'
$ownName = "$($env:COMPUTERNAME.ToLower()).myhomelab.hv.lab"

# Default (non-SNI) binding, catches localhost and anything else
netsh http add sslcert ipport=0.0.0.0:443 certhash=$thumb `
    appid='{00000000-0000-0000-0000-000000000000}' certstorename=MY
New-WebBinding -Name 'Default Web Site' -Protocol https -Port 443 -HostHeader '' -SslFlags 0
(Get-WebBinding -Name 'Default Web Site' -Protocol https -Port 443 -HostHeader '').AddSslCertificate($thumb, 'My')

# Own-name SNI binding
New-WebBinding -Name 'Default Web Site' -Protocol https -Port 443 -HostHeader $ownName -SslFlags 1
(Get-WebBinding -Name 'Default Web Site' -Protocol https -Port 443 -HostHeader $ownName).AddSslCertificate($thumb, 'My')

# Load-balanced-name SNI binding, the one the proxy actually routes to
New-WebBinding -Name 'Default Web Site' -Protocol https -Port 443 -HostHeader 'scom-console.myhomelab.hv.lab' -SslFlags 1
(Get-WebBinding -Name 'Default Web Site' -Protocol https -Port 443 -HostHeader 'scom-console.myhomelab.hv.lab').AddSslCertificate($thumb, 'My')
```

If you haven't already fixed IIS's other Windows-Integrated-Auth
prerequisites (NTLM loopback exemption, `redirection.config` ACL for
`IIS_IUSRS`, etc.), that's a separate checklist, not specific to HA, but
do it on every node, not just the first one.

### 18c. Install and configure HAProxy on pfSense

1. **System → Package Manager → Available Packages** → install
   `pfSense-pkg-haproxy`.

2. **Firewall → Virtual IPs → Add**: Type `IP Alias`, on the interface
   your management servers live on, address = your chosen VIP (pick an
   unused static-range address, verify with a ping test first).

3. **Firewall → Aliases → Add**: a host alias for the VIP (e.g.
   `Console_VIP`), makes the firewall rule in 18d readable.

4. **Services → HAProxy → Backend → Add**:
   - Servers: all 3 nodes, address + port 443, **Encrypt(SSL): no** on
     each (this is a passthrough proxy; HAProxy never originates its
     own TLS connection to the backend, so this stays off)
   - Balance: `Source` (client-IP hash): TCP mode can't do cookie-based
     stickiness since it never reads HTTP, and if your backends don't
     share session state, IP-hash affinity keeps a user pinned to one
     node instead of bouncing them mid-session
   - **Health check method: `Basic` (plain TCP connect), not `SSL`.**
     The `SSL` health-check option in this package uses an SSLv3
     ClientHello. SSLv3 is disabled by default on any reasonably
     current Windows Server, so this check will mark every backend
     permanently DOWN even though they're completely healthy. We lost
     real time to this exact trap; save yourself the debugging.
   - **Check frequency: set an explicit value (e.g. `2000` ms); do not
     leave it blank.** The field's own help text says "for TCP no check
     will be performed if left empty," which means an empty field
     silently disables health checking entirely rather than using a
     default.

5. **Services → HAProxy → Frontend → Add**:
   - External address: the VIP, port 443
   - **Type: `tcp`** (plain TCP mode). This package also offers
     `ssl / https(TCP mode)`, which adds SNI-based ACL routing on top of
     TCP passthrough, useful if you want one VIP serving multiple
     distinct hostnames to different backend pools, but it requires an
     explicit ACL to route anywhere; without one, connections that don't
     match any ACL get dropped rather than falling through to the
     default backend. For a single-hostname setup, plain `tcp` avoids
     that trap entirely.
   - Default Backend: the pool from Step 18.
   - SSL Offloading: leave unchecked (this is what makes it passthrough;
     checking it would terminate TLS here, defeating the entire point)

6. **Services → HAProxy → Settings**: **check "Enable HAProxy"**, easy
   to miss, and the service simply won't run without it, which looks
   from the outside like a routing failure rather than "the proxy never
   started." Also required: **Maximum connections** (a required field,
   despite the description implying otherwise); `100` is generous
   overhead for a handful of concurrent admin users, at roughly 50KB per
   connection.

7. Click **Save** and **Apply Changes** after *every* rule/setting
   change: pfSense stages changes and won't activate them until you
   explicitly apply. If something that should work doesn't, this is the
   first thing to check.

### 18d. Firewall rules

Nothing routes traffic to a newly created VIP by default; you need
explicit pass rules, per source network you want to grant access from:

- **Firewall → Rules → [interface] → Add**: Action `Pass`, Protocol
  `TCP`, Source = that network (or a specific host/alias if you want to
  scope more tightly than "the whole subnet"), Destination = your VIP
  alias, port 443.

Repeat per network/VLAN you want to reach the console, including the
VIP's own subnet, if you don't already have a broad allow rule there.
Don't reflexively open every network you have; scope this to what
actually needs it.

### 18e. DNS

Point your load-balanced hostname (`scom-console.myhomelab.hv.lab`) at the VIP,
not at any individual node.

### 18f. Verify

```powershell
# From a client machine, direct to one node (sanity check, should return
# a 401 Unauthorized, the normal Windows-Auth challenge, not an error):
Invoke-WebRequest https://opsmgr01.myhomelab.hv.lab/OperationsManager/ -SkipCertificateCheck

# Through the proxy, should return the identical 401:
Invoke-WebRequest https://scom-console.myhomelab.hv.lab/OperationsManager/ -SkipCertificateCheck
```

If the direct request succeeds but the proxied one fails with a TLS-layer
error (connection reset mid-handshake, not a clean HTTP error), that's
the signature of the backend health check failing and HAProxy having
nowhere to route to. Check the HAProxy Frontend/Backend status page in
pfSense for a "server is DOWN" warning before assuming it's a network
problem.

Finally, from a machine with a real smartcard/YubiKey PIV credential,
open the console through the load-balanced name and confirm the
certificate prompt appears and authenticates normally; this is the
actual point of the whole exercise, and since the proxy never touches
the TLS session, it should behave identically to hitting any individual
node directly.

**One more real test, not just a config check**: stop the console
service (or just IIS) on one backend node, confirm the proxy's health
check marks it down and traffic keeps flowing through the remaining
nodes without a user-visible failure, then bring it back and confirm it
rejoins the pool.

![The load-balanced web console, signed in via Windows Auth]({{ '/assets/img/gallery/scom-console-load-balanced-dashboards.png' | relative_url }})
_`scom-console.myhomelab.hv.lab`, from a browser on WKS01, Windows Auth
sign-in, and every dashboard from Step 7 still listed and reachable
through the proxy_

## Division of labor

The agent: the entire single-server build (install, agent deployment via
both MECM and by hand, the six-part web console sign-in fix, functional
groups, management pack imports, dashboards, the SQL discovery gMSA trap,
WAAM synthetic monitoring, PIV/smartcard IIS configuration, the SSRS
install and troubleshooting), then the HA work's agent load
redistribution, notification-resilience verification, monitoring-group
coverage fix, and issuing/installing the shared certificate and IIS
bindings on all three nodes. That includes the two hardest bugs in this
whole build: root-causing why the SSRS catalog databases were never
AG-joined, working through the `RSExecRole`/`db_owner`/`sysadmin`
permission ladder and a self-inflicted Unattended Execution Account
break, then actually joining the databases to the AG for real; and,
separately, tracing the Application Advisor/Diagnostics failures back
to a machine-account login missing on two SQL nodes and a SCOM user
role that had sat empty since original setup. The SSRS fix specifically
needed a WMI-permission grant that a safety classifier here blocks
outright, so that piece ran on a second instance with different tool
access, working from a handoff document and reporting back with its own
evidence; the Application Advisor/Diagnostics fix followed from that
same instance once it already had a foothold on the box. Both sagas
were verified independently in this session before being written up
here, not taken on faith. Me: the pfSense/HAProxy
build itself, the VIP, backend, frontend, and firewall rules, since
pfSense stays hands-off for the agent by standing policy no matter what
a task needs, confirming `Set-SCOMLicense`'s actual fix, actually
sitting at the console to click through the RSConfigTool wizard once
the fix needed a genuine interactive GUI session, and the physical
smartcard-reader verification that no script can drive or
inspect.

## What's next

- Automatic failover for the management servers themselves is only as
  good as your last test of it; repeat the stop-a-node test
  periodically, not just once after building this.
- If you want this VIP to eventually front more than one hostname
  (multiple internal apps sharing the same load balancer), revisit the
  `ssl / https(TCP mode)` frontend type with proper SNI ACLs; the
  capability is there, it just wasn't needed for this single-hostname
  setup.
- Consider whether your reverse proxy itself needs its own resilience
  story: right now it's a single point of failure for the load-balanced
  path, even though the 3 backends behind it are redundant.

Up to this point, alerting has meant the console: real, but only for
whoever happens to be looking at it. Email notification comes next,
once there's actually somewhere internal to send it. That means the
first Linux VM in this environment, standing up Postfix, Dovecot, and
Roundcube for real internal email with AD-integrated sign-in, then
giving SCOM's notification channel a real mailbox to send to instead
of depending on anything outside the lab.

## If something doesn't match this guide

Every "why" explanation in this post came from checking the actual system
state directly rather than trusting an assumption: a SQL query against the
OperationsManager database, `Invoke-WebRequest` against the real endpoint,
`Get-SCOMClassInstance` against the real management group. If a step here
doesn't produce what it says it should in your environment, that's the
right move: stop, query what the system actually did, and adjust from
there rather than assuming the guide (or your first hypothesis about what
went wrong) is exactly right.
