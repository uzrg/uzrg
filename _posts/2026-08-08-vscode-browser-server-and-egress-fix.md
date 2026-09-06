---
title: "Homelab Build-Out — Browser VS Code Works, Mostly: Per-User Isolation, a Buried Firewall Rule, and a WebSocket Issue That Won't Stay Fixed"
author: uzrg
date: 2026-08-08 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Microsoft, HyperV, Windows Server]
tags: [Microsoft, VS Code, IIS, PKI, Networking, pfSense, AI Agent]
pin: false
mermaid: false
---

# A browser IDE, and a firewall doing exactly what it was told

Between the last post and this one, the RDS farm finished its base
build: a certificate from the lab's own CA, and two RemoteApp
collections publishing the ConfigMgr console, Notepad++, and Windows
Terminal.

This post covers what came right after: standing up a
browser-accessible VS Code instance on DEVOPS01, in full step-by-step
detail. This build dealt with a string of outbound-internet failures
called "flaky WAN/NAT" across four separate attempts. All four turned
out to be the same single missing outbound firewall rule, working
exactly as designed the whole time.

**Bottom line:** `https://code.myhomelab.hv.lab` is live, Microsoft's
own `code serve-web`, fronted by IIS, bound to a certificate from the
lab's internal CA, with a fully isolated instance per user (Administrator
and test.labuser here, own port, own extensions, own settings) sharing
that one hostname through per-user URL paths. It isn't fully solid
yet: a WebSocket connection still drops mid-session on occasion, an
open problem covered honestly at the end rather than glossed over.
Below is the full build, granular enough that anyone reasonably
comfortable with Windows Server and IIS should be able to reproduce it
end to end for a similar tool, not just this one, followed by the
networking mystery that delayed it.

## What you need before starting

- A domain-joined Windows Server box to host it (this build used
  DEVOPS01, Windows Server 2025).
- An internal Enterprise CA already issuing certificates to domain
  members; this build reused a CA and template set up during earlier
  PKI work, not something stood up fresh here.
- Local admin on the host VM.
- The VS Code CLI (comes bundled with the normal desktop installer, no
  separate download needed) and Git for Windows.
- Two IIS add-ons that don't ship in-box: **Application Request
  Routing** (ARR) and **URL Rewrite**, both free downloads from
  Microsoft.
- Outbound internet access from the host VM on ports 80/443. This
  build hit a wall here that turned out to be the actual story; see
  the second half of this post before assuming your network has it.

The shape of this build is worth naming up front, because it's the
same shape you'd use for almost any local-only web tool that needs to
be reachable securely: **the tool itself stays bound to loopback and
speaks plain HTTP, IIS terminates TLS and reverse-proxies to it, and a
scheduled task keeps the tool running since it has no native Windows
service mode.** Swap `code serve-web` for a different tool and Steps 3,
4, 5, and 7 barely change.

## Step 1: Install the VS Code CLI and Git for Windows

`code serve-web` is a subcommand of the same `code` CLI the desktop
installer drops on `PATH`; there's no separate server-only download to
find. Run the standard installer silently:

```powershell
Start-Process -FilePath '.\VSCodeSetup-x64-<version>.exe' `
    -ArgumentList '/VERYSILENT /NORESTART /MERGETASKS="!runcode,addtopath"' `
    -Wait
```

`!runcode` stops it from trying to launch the GUI editor after install
(there's no interactive desktop session on a server); `addtopath` gets
`code` and `code.cmd` onto `PATH` for every later step.

Then Git for Windows, the same way. The editor's built-in Git
integration needs a real `git.exe` on the box to actually do anything;
it isn't bundled:

```powershell
Start-Process -FilePath '.\Git-<version>-64-bit.exe' `
    -ArgumentList '/VERYSILENT /NORESTART /NOCANCEL /SP- /CLOSEAPPLICATIONS /RESTARTAPPLICATIONS /COMPONENTS="icons,ext\reg\shellhere,assoc,assoc_sh"' `
    -Wait
```

**Verify both landed before moving on.** Don't trust the installer
process's exit code alone: both of these installers spawn a wrapper
process that can sit around after the real work is done without
signaling that it's finished, especially over a remote PowerShell
session. Check for the actual files instead:

```powershell
Test-Path 'C:\Program Files\Microsoft VS Code\bin\code.cmd'
Test-Path 'C:\Program Files\Git\bin\git.exe'
```

Both should return `True`. If a wrapper process (`vscode-setup.exe` /
`git-setup.exe` or similar) is still sitting in `Get-Process` a minute
later even though `Test-Path` already came back `True`, it's safe to
`Stop-Process -Force` it; the install already finished.

## Step 2: Install extensions

Pick whatever extensions the team actually needs; this build installed
four: PowerShell, Python, YAML, and Git Graph.

```powershell
$code = 'C:\Program Files\Microsoft VS Code\bin\code.cmd'
foreach ($ext in @('ms-vscode.powershell','ms-python.python','redhat.vscode-yaml','mhutchie.git-graph')) {
    & $code --install-extension $ext --force
}
```

**If this hangs or times out**, don't assume it's a fluke; see the
firewall section below before retrying blindly. If the host genuinely
has no outbound path to the Marketplace yet, there's a workaround:
download the `.vsix` package for each extension directly from
Microsoft's gallery API from a machine that *does* have internet (a
jump box, your own workstation, whatever's handy):

```
https://marketplace.visualstudio.com/_apis/public/gallery/publishers/<publisher>/vsextensions/<name>/latest/vspackage
```

For example, PowerShell: `publishers/ms-vscode/vsextensions/powershell/latest/vspackage`.

**One trap here worth knowing about:** that endpoint serves the file
gzip-encoded. Downloading it with plain `curl` (no `--compressed`
flag) saves the raw gzip stream instead of a usable `.vsix`, and it
fails on install with `Error: End of central directory record
signature not found. Either not a zip file, or file is truncated.` You
can tell which you've got from the first two bytes: a real zip-based
`.vsix` starts with `PK`; a still-gzipped file starts with `1f 8b`. If
you hit that, decompress it once (`gunzip` the file after renaming it
`.gz`) and it'll extract fine. Then install offline instead of from
the Marketplace:

```powershell
& $code --install-extension 'C:\path\to\ms-vscode.powershell.vsix' --force
```

Confirm what actually landed:

```powershell
& $code --list-extensions
```

## Step 3: Install IIS, ARR, and URL Rewrite

`code serve-web` only binds to a plain local port; it has no TLS, no
virtual-host support, nothing you'd expose directly to a browser. IIS
sits in front of it as a reverse proxy, the same role it would play for
any other tool that only speaks plain local HTTP.

```powershell
Install-WindowsFeature -Name Web-Server, Web-Mgmt-Console, Web-Mgmt-Tools -IncludeManagementTools
```

Then the two add-ons, both plain MSIs, both silent-installable:

```powershell
Start-Process msiexec.exe -ArgumentList '/i ARR3_x64.msi /quiet /norestart' -Wait
Start-Process msiexec.exe -ArgumentList '/i URLRewrite2_x64.msi /quiet /norestart' -Wait
```

Grab both directly from Microsoft; search "Application Request Routing
download" and "URL Rewrite download." **Microsoft rotates the direct
download links periodically**, so if a specific `download.microsoft.com`
URL you find in an old blog post (including this one, eventually) 404s,
go through the current IIS.net download page instead of guessing at a
newer GUID.

**If you're doing this by hand in IIS Manager instead of scripting
it:** after both installs, open IIS Manager, click the top-level
server node in the left tree (not a specific site), and you should see
two new icons in the middle pane: **Application Request Routing
Cache** and **URL Rewrite**. If either is missing, the install didn't
actually register; re-run the MSI and check its log rather than
assuming it worked.

**Verify both modules actually registered:**

```powershell
Import-Module WebAdministration
Get-WebGlobalModule | Where-Object Name -in 'ApplicationRequestRouting','RewriteModule'
```

Both names should come back. If `ApplicationRequestRouting` seems to
be missing, check the *whole* module list rather than a narrow filter
first; a too-specific `-like` pattern can make it look uninstalled
when it isn't.

## Step 4: Enable the ARR proxy and add per-user reverse-proxy rules

By default ARR is installed but its actual proxy behavior is switched
off. Turn it on:

```powershell
Set-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST' `
    -filter 'system.webServer/proxy' -name 'enabled' -value 'True'
```

A single shared `code serve-web` instance works for one person. The
moment a second person needs their own extensions, settings, and open
files without stepping on someone else's, the answer is one isolated
instance per user: its own port, its own `--server-base-path`, and a
matching pair of IIS rewrite rules on the Default Web Site. This build
supports two users, `Administrator` and `test.labuser`, so the rules
exist in pairs, and the pattern is written as a function so adding a
third user later is one more function call, not four more manual rule
edits:

```powershell
function Add-VSCodeReverseProxyRules {
    param([string]$UserName, [int]$Port)

    $site = 'MACHINE/WEBROOT/APPHOST/Default Web Site'
    $path = $UserName.ToLower()

    # Reverse proxy: /<path>/anything -> http://localhost:<port>/<path>/anything
    Add-WebConfigurationProperty -pspath $site -filter 'system.webServer/rewrite/rules' `
        -name '.' -value @{name = "ReverseProxy-$UserName"}
    $proxyFilter = "system.webServer/rewrite/rules/rule[@name='ReverseProxy-$UserName']"
    Set-WebConfigurationProperty -pspath $site -filter "$proxyFilter/match" -name 'url' -value "^$path(.*)"
    Set-WebConfigurationProperty -pspath $site -filter "$proxyFilter/action" -name 'type' -value 'Rewrite'
    Set-WebConfigurationProperty -pspath $site -filter "$proxyFilter/action" -name 'url' -value "http://localhost:$Port/$path{R:1}"

    # Trailing-slash redirect: /<path> (no slash) -> /<path>/
    Add-WebConfigurationProperty -pspath $site -filter 'system.webServer/rewrite/rules' `
        -name '.' -value @{name = "NormalizeTrailingSlash-$UserName"}
    $slashFilter = "system.webServer/rewrite/rules/rule[@name='NormalizeTrailingSlash-$UserName']"
    Set-WebConfigurationProperty -pspath $site -filter "$slashFilter/match" -name 'url' -value "^$path`$"
    Set-WebConfigurationProperty -pspath $site -filter "$slashFilter/action" -name 'type' -value 'Redirect'
    Set-WebConfigurationProperty -pspath $site -filter "$slashFilter/action" -name 'url' -value "/$path/"
    Set-WebConfigurationProperty -pspath $site -filter "$slashFilter/action" -name 'redirectType' -value 'Found'
}

Add-VSCodeReverseProxyRules -UserName 'Administrator' -Port 8001
Add-VSCodeReverseProxyRules -UserName 'test.labuser' -Port 8002
```

**The trailing-slash rule isn't optional decoration.** `code serve-web`
serves its web UI assuming its own address ends in `/`, so every
relative script and stylesheet URL it emits resolves against that
base. Hit `/administrator` without the trailing slash and the browser
resolves those relative paths against the site root instead of
`/administrator/`, which loads a blank page with a wall of 404s in the
browser console and looks exactly like a broken deployment. The
redirect rule exists purely to force that slash before the browser
ever asks for an asset.

**GUI equivalent**, if you'd rather click through it for each user:
IIS Manager, select **Default Web Site**, double-click **URL Rewrite**,
**Add Rule(s)**, **Blank rule**. For the proxy rule: Pattern
`^username(.*)`, Action type Rewrite, Rewrite URL
`http://localhost:<port>/username{R:1}`. For the trailing-slash rule:
Pattern `^username$`, Action type Redirect, Redirect URL `/username/`,
Redirect type Found (302).

**Verify the rules landed correctly:**

```powershell
Get-WebConfiguration -pspath 'MACHINE/WEBROOT/APPHOST/Default Web Site' -filter 'system.webServer/rewrite/rules/rule' | Select-Object Name
```

You should see all four: `ReverseProxy-Administrator`,
`NormalizeTrailingSlash-Administrator`, `ReverseProxy-test.labuser`,
`NormalizeTrailingSlash-test.labuser`.

## Step 5: Request and bind a certificate from the internal CA

If your CA already has a template that allows a manually-specified SAN
at request time (this build reused one called `WebServerManualSAN`,
built earlier for a different service on the same CA; the stock
`WebServer` template that ships by default often only builds SAN
entries from AD automatically, which won't let you request an
arbitrary hostname like this one), request a cert:

```powershell
$infPath = 'C:\Temp\code-cert.inf'
@"
[Version]
Signature="`$Windows NT`$"

[NewRequest]
Subject = "CN=code.myhomelab.hv.lab"
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
"@ | Out-File -FilePath $infPath -Encoding ascii

certreq -new -f $infPath 'C:\Temp\code-cert.req'
certreq -submit -f -attrib "CertificateTemplate:WebServerManualSAN`nSAN:dns=code.myhomelab.hv.lab" `
    -config "DC01.myhomelab.hv.lab\myhomelab-DC01-CA" 'C:\Temp\code-cert.req' 'C:\Temp\code-cert.cer'
certreq -accept -machine -f 'C:\Temp\code-cert.cer'
```

**Two easy mistakes here**, both worth calling out explicitly for
anyone following along for the first time:

- The final `-accept` step needs `-machine` explicitly, because the
  request used `MachineKeySet = TRUE`. Leave it off and you'll get
  `Expected -user | -machine argument` instead of a clean install.
- If your default template only supports AD-built SANs (check
  `msPKI-Certificate-Name-Flag` on the template object: a value of
  `402653184` means AD-built, `1` means you can supply one manually),
  a request like this one will get rejected with
  `CERTSRV_E_SUBJECT_DNS_REQUIRED` no matter how you supply the SAN.
  You'll need a template that actually allows a manual SAN before this
  step will work at all (that's outside the scope of this post), but
  don't spend an hour testing different `-attrib` syntaxes if this is
  the actual reason; one setting on the template is the real fix.

Confirm the cert landed in the machine store:

```powershell
Get-ChildItem Cert:\LocalMachine\My | Where-Object Subject -like '*code.myhomelab.hv.lab*'
```

Note the `Thumbprint` it returns; you need it for the next step.

Bind it to the site on port 443:

```powershell
Import-Module WebAdministration
$thumb = '<paste the thumbprint here, no spaces>'

New-WebBinding -Name 'Default Web Site' -Protocol https -Port 443 -HostHeader 'code.myhomelab.hv.lab'
(Get-WebBinding -Name 'Default Web Site' -Protocol https).AddSslCertificate($thumb, 'my')
```

**GUI equivalent**: IIS Manager, **Default Web Site**, **Bindings** in
the right-hand actions pane, **Add**. Type `https`, Host name
`code.myhomelab.hv.lab`, and pick the certificate by its friendly name
(usually the CN, `code.myhomelab.hv.lab`) from the dropdown.

## Step 6: DNS

One A record, on whichever DNS server your domain's clients actually
use, `code.myhomelab.hv.lab` pointing at the host VM's IP. If you're
on AD-integrated DNS, this is a two-minute change on a domain
controller; nothing special about it.

## Step 7: Run `code serve-web` persistently, per user, via Scheduled Tasks

`code serve-web` itself is just a foreground process; it has no native
Windows service mode. A Scheduled Task set to run at startup, as
SYSTEM, with a restart policy, is the simplest way to keep it running
without adding a third-party service wrapper. This pattern, a
Scheduled Task standing in for a real Windows service, is worth
keeping in your back pocket generally: it comes up constantly for
CLI-only tools that were never packaged as a proper service.

Each user gets their own task, own data directory, own connection
token, and own port, matching the base path wired up in Step 4. A
single function builds one user's isolated instance end to end:

```powershell
function New-VSCodeServeWebInstance {
    param([string]$UserName, [int]$Port)

    $codeExe = 'C:\Program Files\Microsoft VS Code\bin\code.cmd'
    $path = $UserName.ToLower()
    $dataDir = "C:\VSCodeServer\$UserName"
    New-Item -ItemType Directory -Path $dataDir -Force | Out-Null

    # Don't run with --without-connection-token just because this is an
    # internal box; there's no good reason to skip auth entirely.
    $token = [guid]::NewGuid().ToString('N')
    $tokenFile = Join-Path $dataDir 'connection-token.txt'
    Set-Content -Path $tokenFile -Value $token -NoNewline
    icacls $tokenFile /inheritance:r /grant:r 'SYSTEM:F' 'BUILTIN\Administrators:F' | Out-Null

    $argList = @(
        'serve-web',
        '--host 127.0.0.1',
        "--port $Port",
        "--server-base-path `"/$path`"",
        "--server-data-dir `"$dataDir`"",
        "--connection-token-file `"$tokenFile`"",
        '--accept-server-license-terms'
    ) -join ' '
    $action = New-ScheduledTaskAction -Execute $codeExe -Argument $argList
    $trigger = New-ScheduledTaskTrigger -AtStartup
    $principal = New-ScheduledTaskPrincipal -UserId 'SYSTEM' -LogonType ServiceAccount -RunLevel Highest
    $settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries `
        -StartWhenAvailable -RestartCount 999 -RestartInterval (New-TimeSpan -Minutes 1) `
        -ExecutionTimeLimit ([TimeSpan]::Zero)

    Register-ScheduledTask -TaskName "VSCodeServeWeb-$UserName" -Action $action -Trigger $trigger `
        -Principal $principal -Settings $settings `
        -Description "Browser-accessible VS Code for $UserName (code serve-web), fronted by IIS reverse proxy on 443"

    Start-ScheduledTask -TaskName "VSCodeServeWeb-$UserName"
}

New-VSCodeServeWebInstance -UserName 'Administrator' -Port 8001
New-VSCodeServeWebInstance -UserName 'test.labuser' -Port 8002
```

**GUI equivalent**, per user, through Task Scheduler: General tab,
name it `VSCodeServeWeb-<user>`, run whether user is logged on or not,
highest privileges, configure for Windows Server 2025. Triggers tab,
new trigger, "At startup." Actions tab, "Start a program," program is
`code.cmd`, arguments are everything after `serve-web` above for that
user's port, base path, and data directory. Settings tab, "Restart the
task every" 1 minute with an unlimited restart count, "If the running
task does not end when requested, force stop it," and no execution
time limit (don't let Task Scheduler kill an intentionally
indefinite task after its default 3-day limit).

`--host 127.0.0.1` is deliberate for every instance: each process only
needs to be reachable from IIS on the same box, not directly from the
network. IIS is the only thing that should ever see these ports from
outside, and the per-user `--server-base-path` is what lets IIS tell
the instances apart while they all share the one public port 443.

**Verify each instance is actually listening:**

```powershell
Get-ScheduledTask -TaskName 'VSCodeServeWeb-*' | Select-Object TaskName, State
Get-NetTCPConnection -LocalPort 8001,8002 | Select-Object LocalPort, State
```

Both tasks should show `Running`, and both ports `Listen`.

## Step 8: First run and verification

Open `https://code.myhomelab.hv.lab/administrator/?tkn=<the token from
that user's connection-token.txt>` in a browser (or `/test.labuser/`
with that user's own token; each path only ever accepts the token that
was generated for it). What you should see, in order:

1. **"The latest version of the Visual Studio Code Server is
   downloading, please wait a moment..."** with an auto-refreshing
   page. This is normal on first run; it's pulling the actual server
   component down from Microsoft. Give it a minute; if it never
   clears, see the troubleshooting section below.
2. A redirect that drops the `?tkn=...` from the URL and sets that
   same token as a cookie instead. Also normal; see the Security
   notes section below for exactly what that cookie is.
3. The actual VS Code web UI.

![Administrator's isolated instance, loaded at /administrator/]({{ '/assets/img/gallery/vscode-browser-administrator-instance.png' | relative_url }})
_Administrator's session: own tab, own path, nothing shared with test.labuser_

![test.labuser's instance, loaded side by side with Administrator's own tab]({{ '/assets/img/gallery/vscode-browser-testlabuser-instance.png' | relative_url }})
_Both users' instances open at once, from the same browser, proving the path-based isolation actually holds_

After that first successful load, the session cookie means you don't
need the token in the URL again for a while (it's set for 30 days).
Because the cookie is scoped to its path (`/administrator/` or
`/test.labuser/`), each user's session lives independently of the
other's; one user being logged in never grants any access to the
other user's instance. **Browsing to either path with no token and no
existing cookie will correctly get rejected with a 403**; that's the
app itself refusing an unauthenticated request, not a
misconfiguration. Don't chase that as a bug if it's the very first
thing you try.

## Troubleshooting checklist

If the page never gets past step 1 above (stuck downloading, or
eventually shows a plain `500` error instead), work through this list
in order rather than guessing. This same order, backend reachability
first, then proxy config, then client quirks, applies to almost any
reverse-proxied web service, not just this one.

- **Can this VM reach the internet on 443 at all?** Run
  `Invoke-WebRequest https://update.code.visualstudio.com/api/latest/server-win32-x64-web/stable`
  directly on the box. `code serve-web` calls that URL on every page
  load and hard-fails the whole page if it can't reach it; this isn't
  a background check you can ignore.
- **If that request fails, don't assume it's a random network blip
  before checking your firewall's actual egress rules for this
  subnet.** This build lost real time treating a deliberate,
  working-as-designed default-deny as intermittent flakiness; see the
  second half of this post. A server VM that's never been explicitly
  granted outbound internet access will fail this every single time,
  which can look identical to "flaky" if you only retry instead of
  checking what's actually allowed.
- **Confirm the IIS rewrite rule and ARR proxy setting are both
  actually active.** Check `Get-WebConfigurationProperty` for
  `system.webServer/proxy`'s `enabled` value, and re-check the rule
  from Step 4.
- **Confirm the specific user's `code serve-web` instance is actually
  listening locally** before blaming IIS: `curl http://127.0.0.1:8001/`
  for Administrator, `curl http://127.0.0.1:8002/` for test.labuser,
  directly on the host, bypassing the reverse proxy entirely, isolates
  whether the problem is that user's backend or the proxy in front of
  it. Check the right port for the user you're actually testing; a
  clean answer on one user's port says nothing about the other's.
- If you're testing with `curl.exe` from Windows and get a connection
  that hangs or fails instantly with no useful error against the HTTPS
  hostname specifically (while everything else works), try
  `Invoke-WebRequest` instead before assuming IIS is broken. Curl's
  Windows TLS backend (schannel) has been seen to choke on
  renegotiation against some IIS bindings for reasons unrelated to the
  actual site config.

## Check the ruleset before the theories: a firewall rule working exactly as designed

The Marketplace timeout back in Step 2, and the endless retries
against `update.code.visualstudio.com` in Step 8, were the same
underlying problem showing up twice on DEVOPS01. It looked like
unpredictable "WAN/NAT flakiness," the kind of thing that comes and
goes and never quite gets root-caused because it never fails the same
way twice. It had actually failed the exact same way every single
time; nobody had lined up *which host* was being tested against
*which one* actually worked.

Reading pfSense's actual ruleset for VLAN10, the subnet every server
in this lab lives on, settled it in a few minutes: there was no
general outbound-internet rule for that subnet at all. Exactly one
host, WSUS01, had an explicit allow rule out to the internet on
80/443, because WSUS is the only thing that's supposed to reach the
internet directly. Every WAN timeout on every other VM, including
DEVOPS01, had been that same design working exactly as intended,
mistaken for a bug because nobody had checked.

The fix was one rule mirroring the one that already worked: a new host
alias for DEVOPS01's IP, reusing the existing `MSUpdate_Ports` alias
(80/443), one pass rule on the VLAN10 interface allowing that host out
to anywhere on those ports. Deliberately not scoped down to a specific
list of allowed hostnames, even though the pieces for that existed in
the config already (an unused `MSUpdate_FQDNs` alias); the proven
WSUS01 rule doesn't use one either, suggesting that approach was tried
and abandoned, likely because CDN-backed endpoints rotate IPs faster
than a periodically-refreshed hostname-based allow-list keeps up.

With that rule in place, every symptom above cleared immediately on
DEVOPS01; no other change needed.

## Security notes: how tokens actually work here

There's no admin console and no `code` subcommand for looking up,
resetting, or rotating a user's connection token; checked the full
`code --help` and `code serve-web --help` output directly to confirm
neither exists. The token is just a GUID, generated once by the script
in Step 7 and written to a plain text file:

```
C:\VSCodeServer\Administrator\connection-token.txt
C:\VSCodeServer\test.labuser\connection-token.txt
```

`code serve-web` reads that file on startup and matches whatever comes
in on `?tkn=` against it.

**That file isn't the only place the plaintext token lives.** After
the first successful request, the same secret gets handed back to the
browser as a cookie, not exchanged for some separate, opaque session
ID. Confirmed by inspecting the actual cookie jar: the cookie is
literally named `vscode-tkn`, and its value is the same GUID from the
token file, verbatim, alongside two smaller
secondary cookies (`vscode-secret-key-path`, `vscode-cli-secret-half`)
that ride along with it. The practical consequence: anyone who can
read that user's browser cookie store for the 30-day life of that
cookie can recover the plaintext connection token, not just someone
with sufficient rights on DEVOPS01 running `Get-Content` against the
server-side file. Two paths to the same plaintext secret, not one.

The whole lifecycle, worth having in one place rather than pieced
together from Steps 7 and 8: the token is generated once in Step 7 and
written to that file; a user's first request supplies it via `?tkn=`;
the server sets that same value as the `vscode-tkn` cookie and drops
it from the URL; every request after that rides the cookie, valid 30
days; rotating writes a new GUID into the same file, which invalidates
every cookie issued against the old value immediately once the running
process actually picks up the change, since the cookie is checked
against the same file on every request, not a separate session store.

Three operational consequences follow from that:

- **Rotating a token is a manual, disruptive operation, and "restart
  the task" isn't the same guarantee as "restart the process."**
  Confirmed live during this review: `Stop-ScheduledTask` followed by
  `Start-ScheduledTask` left the existing `code-tunnel.exe` running
  untouched, still holding the old token in memory and
  silently ignoring the new value written to the file. It kept
  accepting TCP connections on the port without ever completing a
  response, a hang, not a clean rejection, until it was found and
  killed directly by PID. Deleting the token file first, instead of
  writing a new value, has its own separate failure mode: the task
  reports success, but the underlying process doesn't come back
  cleanly, and the port is left showing `Listen` against a process
  that no longer exists. The version that actually held up, verified
  live more than once against test.labuser's own instance: write the
  new token first, kill the real process by PID rather than trusting
  the task state, then restart:

  ```powershell
  $taskName = 'VSCodeServeWeb-test.labuser'
  $tokenFile = 'C:\VSCodeServer\test.labuser\connection-token.txt'

  Stop-ScheduledTask -TaskName $taskName
  Get-CimInstance Win32_Process -Filter "Name='node.exe' or Name='code-tunnel.exe'" |
      Where-Object { $_.CommandLine -match [regex]::Escape('/test.labuser') } |
      ForEach-Object { Stop-Process -Id $_.ProcessId -Force }

  $newToken = [guid]::NewGuid().ToString('N')
  Set-Content $tokenFile -Value $newToken -NoNewline
  Start-ScheduledTask -TaskName $taskName
  ```

  Once the old process is actually gone, rotation invalidates every
  existing session cookie for that user's path immediately; there's no
  "sign out everywhere else" button beyond that.
- **The plaintext token is recoverable from two places, not one: the
  server-side file, and any browser that's ever held a live session
  cookie.** Each token file is ACL'd to `SYSTEM` and
  `BUILTIN\Administrators`, not to the individual user it belongs to,
  because the scheduled tasks run as SYSTEM, not as that user. In
  practice, that means anyone who already has local administrator
  rights on DEVOPS01 can read any user's token from the file, not just
  their own, the same way local admin can already read cached
  credentials or reset any local password. That's a lab-acceptable
  trade-off, the same category as the CA living on a domain controller
  back in Phase 2. It's no longer the *only* trade-off, though: since
  the cookie carries that same plaintext value, whoever controls that
  user's browser profile for the 30-day cookie life holds the same
  secret too, worth factoring in before assuming the server-side ACL
  is the whole story.
- **The isolation between users is enforced by IIS routing and
  separate processes, not by an OS-level permission boundary.** Fine
  here because everyone with admin on this box is already fully
  trusted, and not something to carry forward unexamined into a build
  where the users on one shared host genuinely shouldn't be able to
  reach each other's data.

If this pattern ever needs to hold up against untrusted users sharing
one host, the fix isn't a bigger token, it's running each instance
under that user's own account instead of SYSTEM, with an ACL that
actually excludes the other users, and moving the secret out to a
proper secrets store instead of a flat file and a plaintext cookie.
Neither was warranted for two trusted admins on a lab box, but it's
the right escalation path to know about before this pattern gets
reused somewhere with a real multi-tenant boundary.

## Lessons learned

- **"Intermittent" often just means "different machines, never
  compared."** Every failed attempt was 100% consistent on the
  same machine, every time. It only looked random because separate
  tests hit different VMs without anyone lining up the results.
  Before calling something flaky, confirm you're testing the same
  source against the same destination each time.
- **Read the actual firewall ruleset for the specific interface
  involved before diagnosing a network as broken.** NAT modes, gateway
  health, and installed security packages are all reasonable first
  guesses, but none of them matter until the simplest explanation,
  nothing was ever allowed out from this host, gets ruled out. Check
  the ruleset before the theories.
- **`code serve-web` will not work at all on a machine without
  outbound internet access, full stop.** It calls
  `update.code.visualstudio.com` on every page load and treats a
  failed call as fatal to the whole request, not a background warning
  that degrades gracefully. An internal, browser-only tool that only
  needs to be reachable on the domain still needs its own path out to
  the internet just to function; **confirm that access exists before
  promising anyone this will "just work"** on a network with
  restricted egress.

## Division of labor

The agent: the full IIS/ARR/cert/DNS/scheduled-task build, the
extension installs and the gzip fix, the firewall config review that
narrowed the cause down to one interface's ruleset, live token-rotation
testing that uncovered three separate failure modes (the delete-first
bug, a task restart that left the old process running untouched and
hung, and the discovery that the "session cookie" is just the raw
token echoed back rather than a distinct session ID), catching and
fixing the orphaned-worker recurrence a second time during final
review, and the WebSocket investigation that found the disabled ping
interval and shipped both that fix and the periodic cleanup task. Me:
the actual insight that ended the firewall investigation, that only
one specific machine was ever meant to reach the internet, applying
the firewall change itself since the firewall stays hands-off for the
agent by standing policy no matter what a task seems to need, and the five specific questions (WebSocket support
installed, ARR passthrough, keep-alive timeouts, token lifecycle,
update-check hygiene) that actually pointed at the ping interval and
turned this from an unexplained shrug into a real lead.

## What's next

Browser VS Code is live and reachable from anywhere on the
domain-joined network, and both instances load and hold a session
correctly, as shown above. What hasn't held up is a long editing
session: the connection intermittently drops with a WebSocket close
(error code 1006), forcing a reload to get back in.

![Recurring WebSocket close, error code 1006, forcing a full page reload]({{ '/assets/img/gallery/vscode-browser-websocket-error.png' | relative_url }})
_This has recurred more than once, including after adjusting IIS's WebSocket-related timeout settings; the fix so far hasn't stuck_

That's not a solved problem, but it's no longer an unexplained shrug.
Ruled out first: WebSocket Protocol Support, ARR's proxy setting, and
IIS's own WebSocket module are all installed and enabled, the correct
combination for proxying a WebSocket Upgrade at all; and the
connection token can't be the cause, since it's only checked once
during the handshake, the 30-day session cookie is what actually
carries a live session.

The real gap: ARR's own proxy timeout had already been raised once, to
30 minutes, but IIS's WebSocket module still had `pingInterval` set to
`00:00:00`, disabled. With no ping/pong frames going out, an idle
WebSocket has nothing keeping it alive through a NAT table or a
firewall's connection tracking; once that state times out, the next
packet gets reset or dropped, and the browser sees exactly what error
1006 describes, an abnormal closure with no close frame. Fixed by
setting a real ping interval:

```powershell
Set-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST/Default Web Site' `
    -filter 'system.webServer/webSocket' -name 'pingInterval' -value '00:02:00'
```

Two minutes is frequent enough to keep most NAT and firewall state
alive without being wasteful. That section is locked by default at the
server level, the same lock encountered earlier configuring the RD Web
Access work, so it needed `appcmd unlock config
-section:system.webServer/webSocket` first; applied and confirmed live.
This addresses a real gap that was never tried before, but proving it
actually stops the disconnect for good needs a sustained idle session
held open longer than a single review pass allows.

A separate, real contributing factor: `node.exe` genuinely leaves
orphaned generations behind on self-update checks, caught recurring
twice more during this review, two stale generations on
test.labuser's instance and three quietly piled up behind
Administrator's, which had been answering every request fine the whole
time, proof that "it responds" and "it's healthy underneath" are two
different claims. Manually killing orphans and restarting the task
doesn't scale, so a proper mitigation is now deployed instead: a
Scheduled Task, `VSCodeServeWeb-OrphanCleanup`, running every 15
minutes, checking each user's process tree for more than one live
`node.exe` and cleaning up automatically when it finds one:

```powershell
foreach ($UserName in 'Administrator','test.labuser') {
    $path = $UserName.ToLower()
    $taskName = "VSCodeServeWeb-$UserName"
    $pattern = [regex]::Escape("--server-base-path /$path ")
    $matching = Get-CimInstance Win32_Process -Filter "Name='node.exe' or Name='code-tunnel.exe'" |
        Where-Object { $_.CommandLine -match $pattern }
    $nodeCount = ($matching | Where-Object { $_.Name -eq 'node.exe' }).Count

    if ($nodeCount -gt 1) {
        Stop-ScheduledTask -TaskName $taskName -ErrorAction SilentlyContinue
        Start-Sleep -Seconds 2
        $matching | ForEach-Object { Stop-Process -Id $_.ProcessId -Force -ErrorAction SilentlyContinue }
        Start-Sleep -Seconds 2
        Start-ScheduledTask -TaskName $taskName
    }
}
```

Tested against Administrator's actual three-generation pileup before
being scheduled: correctly detected the count, cleaned up, and the
instance came back serving normally. This mitigates the symptom, not
the cause; why `code serve-web`'s own update check leaves the old
generation running instead of replacing it cleanly is still genuinely
unknown. One suggestion rejected along the way: skipping the update
check entirely via a flag. `code serve-web --help` doesn't have one,
worth verifying a claim like that against the tool's own `--help`
before trusting it, the same lesson as the CA template flag earlier in
this build.

Both fixes, the ping interval and the cleanup task, are deployed and
real; neither is proven sufficient on its own. Until an actual long
session survives cleanly, this deployment stays **experimental**, not
something to hand to anyone expecting a production-grade remote
editor.

DEVOPS01 isn't done carrying new weight. Right behind this build comes
a much bigger one on the same box: Azure DevOps Server on-premises,
databases living on the SQL AG, IIS fronting it with a proper
certificate, domain accounts signing in with zero separate passwords
to manage. A browser IDE and a full DevOps platform, running on the
same server in quick succession, are where this lab is heading.

## Quick reference

The one thing to check first for each failure mode, in the order
you're likely to hit them:

| Symptom | Check this first |
|---|---|
| Extension install hangs or times out | `Invoke-WebRequest https://update.code.visualstudio.com/api/latest/server-win32-x64-web/stable` on the host; if that fails, it's the firewall, not VS Code |
| `.vsix` install fails with a zip error | First two bytes of the downloaded file: `PK` is a real package, `1f 8b` is still gzipped and needs decompressing first |
| Page loads blank with a wall of 404s | Did you land on `/administrator` without the trailing slash? Check the `NormalizeTrailingSlash-<user>` rule actually fired |
| `certreq -accept` fails with an argument error | Did you include `-machine`? Required whenever the request used `MachineKeySet = TRUE` |
| Cert request fails with `CERTSRV_E_SUBJECT_DNS_REQUIRED` | Template's `msPKI-Certificate-Name-Flag` isn't `1`; it doesn't allow a manually-specified SAN |
| Page stuck downloading, then a plain `500` | Work the full troubleshooting checklist above in order: backend reachability, then proxy config, then client quirks |
| Scheduled task shows `Running` but the port isn't `Listen`, or requests to it hang | Look for more than one `node.exe` (or a `code-tunnel.exe` wrapper with no worker under it) tied to that user's `--server-base-path`; the `VSCodeServeWeb-OrphanCleanup` task should catch this within 15 minutes on its own, but the same manual kill-and-restart still works immediately |
| WebSocket closes with error 1006 mid-session | Check `system.webServer/webSocket`'s `pingInterval` first, `00:00:00` means no keep-alive frames, so an idle connection can get silently dropped by any stateful device in the path; also confirm there's exactly one `node.exe` bound to that user's `--server-base-path`, more than one live worker is a real, separate condition this build has hit. Neither fix is proven yet to stop the disconnect from recurring for good |
