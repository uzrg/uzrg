---
title: "Homelab Build-Out — Standing Up the RDS Farm, and Making Smart Cards Actually Work"
author: uzrg
date: 2026-08-07 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Microsoft, HyperV, Windows Server]
tags: [Microsoft, RDS, RemoteApp, PKI, Windows Server, AI Agent]
pin: false
mermaid: false
---

# Five servers, one farm, and three problems that weren't what they first looked like

The base deployment covers five servers: a broker, a gateway, a
licensing server, and two session hosts, publishing RemoteApps instead
of full desktops. Three challenges arose during the deployment before
the farm could actually be used, and none of them turned out to be
what they looked like at first: a role-install error that pointed at
the network, then the install media, before the real cause we'll see
later; a
duplicated app icon that looked cosmetic and turned out to be a real
access-control gap; and a smart card sign-in that looked like a single
IIS setting away and needed a full authentication-mode switch instead.

The design behind it: two collections, deliberately generic rather
than named after a single app, since more apps get published into
them later, `RDCOL-Admin` on `RDSSH01` starting with ConfigMgr
Console, `RDCOL-General` on `RDSSH02` starting with Notepad++. One
certificate carries two SAN names, `rdsgw.myhomelab.hv.lab` for the
people actually typing a URL, and `rdcb.myhomelab.hv.lab`, a purely
internal name the broker's HA redirector uses that no person ever
sees. RD Web Access gets a second authentication option, YubiKey PIV
smart card, sitting alongside the password form rather than replacing
it; FSLogix profile containers come after that, on their own dedicated
share.

**Bottom line:** all five RDS roles are up (`RDSCB` as broker, `RDSGW`
as gateway and web access, `RDSLC` as licensing, `RDSSH01` and
`RDSSH02` as session hosts), each RemoteApp collection is publishing
two apps, and the certificate covering both the user-facing gateway
name and the broker's internal HA name is trusted across all four
certificate roles. RD Web Access now accepts a YubiKey PIV smart card
as well as a password, without dropping the password path. Broker HA
against the SQL AG and FSLogix are still ahead, each one waiting on
its own go-ahead before touching a shared system.

## What already existed before this build

Three pieces of earlier work this post leans on, worth naming up front
rather than assuming familiarity. The internal Enterprise CA, its
certificate templates, and the autoenrollment plumbing all came out of
the [Phase 2 PKI build]({% post_url 2026-07-14-phase2-pki-gmsa %});
this build reuses that CA directly. YubiKey PIV smart card logon, the
`SmartcardLogon` certificate
template and the actual certificate `test.labuser` already carries,
came out of a
[separate YubiKey PIV project]({% post_url 2026-07-16-yubikey-piv-smartcard-logon %})
built for domain logon on WKS01; this build reuses that certificate
as-is, no new enrollment needed. And the SQL Always On Availability
Group, `SQLAG01`, listener `SQLAGL01`, already exists from
[its own build]({% post_url 2026-07-17-sql-ag-build %}); it's the
target for Connection Broker HA in What's next below, not something
built here.

What actually gets built fresh in this post: all five RDS roles, both
RemoteApp collections, a new certificate template for the two-SAN
certificate this deployment specifically needs, the access-control
group structure, and the smart card wiring on RD Web Access.

## Cleared before the first cmdlet

Three checks came before touching any RDS cmdlet. First, the CA: a
two-SAN certificate needs a CA that actually honors a
manually-supplied SAN, so confirmed `EDITF_ATTRIBUTESUBJECTALTNAME2`
wasn't set on `myhomelab-DC01-CA` (it would have silently dropped SAN
entries), checkpointed DC01, enabled it via `certutil -setreg
policy\EditFlags +EDITF_ATTRIBUTESUBJECTALTNAME2`, restarted
`CertSvc`, confirmed it stuck. Second, DNS: one A record,
`rdcb.myhomelab.hv.lab` → `10.10.10.26` (RDSCB's static IP), for the
broker's HA client-access name; its PTR record failed to create since
that IP's PTR already points at RDSCB itself, cosmetic only. Third,
NPS01: confirmed not installed, and not a blocker, Gateway's Central
Access Policies run on local policy until NPS01 exists to take them
over.

## The feature-install saga, and an error that pointed at the wrong problem twice

`Install-WindowsFeature -Name RSAT-RDS-Tools` on RDSCB failed with
`0x800f0922`, "source files could not be found," on a fresh, fully
patched deployment. That error chased two wrong theories before
landing on the real one.

The first theory blamed the network. Windows Update as the source
failed outright, unsurprising given this lab's known, still-unexplained
WAN/NAT flakiness: pfSense's own outbound connectivity tested clean,
but traffic forwarded from VLAN10 failed unpredictably, WSUS01
succeeding on one test while DC01, MECM01, and RDSCB failed the same
test seconds apart. A real contributing factor, just not the actual
blocker; it finally got root-caused in the next build in this series
(a missing pfSense firewall rule, not the network itself), covered in
its own post once that one publishes.

The second theory blamed the install media. Explicit local media, a
Windows Server 2025 ISO mounted on all five RDS VMs, failed
identically. Not a version mismatch either: the ISO's `install.wim`
and RDSCB's own build both reported the same `UBR 1742`.

The actual cause was neither: `sources\sxs` on a standard install disc
has never held the Server Roles and Features payload, only a couple
of legacy on-demand packages, confirmed by listing the folder, three
files, none RDS-related. The real payload lives inside `install.wim`
itself, referenced via a `WIM:<path>:<index>` string or a mounted WIM
image, never a raw path to `sxs`. `sxs` looks like the obvious source
folder and isn't one past a handful of legacy components.

The fix: mount `install.wim` index 2 ("Windows Server 2025 Standard,
Desktop Experience") read-only to `C:\WIMMount` on all five VMs, and
point the `HKLM:\SOFTWARE\Policies\Microsoft\Windows\Servicing`
local-source policy at it. An explicit `-Source C:\WIMMount` succeeded
immediately, 15,744 files confirmed in the mounted `WinSxS`. One loose
end: the registry policy alone, without an explicit `-Source`, didn't
pick up the fix reliably when tested standalone, but worked once
invoked through `New-RDSessionDeployment` after a reboot. That
pattern, explicit `-Source` directly, the mounted-WIM policy as
fallback, held for the rest of this build with no repeat of
`0x800f0922`.

One more gate tripped on the first deployment attempt:
`New-RDSessionDeployment` refused to proceed until a pending reboot
cleared on both RDSCB and RDSSH01. Rebooted both, retried, succeeded.

```
RDSCB.MYHOMELAB.HV.LAB   -> RDS-CONNECTION-BROKER
RDSSH01.myhomelab.hv.lab -> RDS-RD-SERVER
```

## Gateway, Licensing, and the second session host

With the WIM lesson in hand, the remaining three roles went in clean:
Windows Features pre-installed explicitly with `-Source C:\WIMMount`
before calling `Add-RDServer` for each, `RDS-Gateway` and
`RDS-Web-Access` on RDSGW, `RDS-Licensing` on RDSLC, `RDS-RD-Server` on
RDSSH02. All three succeeded first try. The same pending-reboot gate
caught RDSGW and RDSLC; RDSSH02 didn't, its own feature install having
already forced one.

```
RDSCB   -> RDS-CONNECTION-BROKER
RDSSH01 -> RDS-RD-SERVER
RDSSH02 -> RDS-RD-SERVER
RDSGW   -> RDS-GATEWAY, RDS-WEB-ACCESS
RDSLC   -> RDS-LICENSING
```

![Server Manager's RDS deployment overview, all 5 roles across 6 servers]({{ '/assets/img/gallery/rds-server-manager-deployment-overview.png' | relative_url }})
_The same result, from Server Manager: RD Connection Broker on RDSCB, Gateway and Web Access sharing RDSGW, Licensing on RDSLC, both session hosts, both collections_

Licensing isn't functional on its own; it needs a mode and a server:
`Set-RDLicenseConfiguration -LicenseServer RDSLC.myhomelab.hv.lab
-Mode PerUser`. Per User over Per Device fits this lab: a handful of
named people reaching RemoteApps from more than one device each suits
user-based CALs better than device-based ones. Confirmed via
`Get-RDLicenseConfiguration`.

## A certificate that needed its own template

The default `WebServer` template couldn't do this. Its
`msPKI-Certificate-Name-Flag` builds SANs from AD automatically, the
right default for most services, wrong here: RDS needs two specific
SAN names, `rdsgw.myhomelab.hv.lab` and `rdcb.myhomelab.hv.lab`,
supplied at request time, something AD can't infer on its own. A
manually-specified SAN got rejected with
`CERTSRV_E_SUBJECT_DNS_REQUIRED` no matter how it was supplied.

The fix: a new template, `WebServerManualSAN`, duplicated from
`WebServer` with `msPKI-Certificate-Name-Flag` set to 1, built via raw
ADSI since no cmdlet creates templates, published to the CA. Requested
a certificate against it with both SAN names, applied it to all four
RDS certificate roles, Gateway, Web Access, Connection Broker
Publishing, Connection Broker Single Sign-On, via `Set-RDCertificate`.
All four Trusted.

This template earns its keep beyond one certificate: it's the standard
answer whenever a future service needs a manually specified SAN, and
gets reused as-is later in this build-out, in
[the browser VS Code server build]({% post_url 2026-08-08-vscode-browser-server-and-egress-fix %}).

## Two RemoteApp collections, and the app with no .exe to point at

Built both collections with `New-RDSessionCollection`, `RDCOL-Admin`
on RDSSH01, `RDCOL-General` on RDSSH02. Publishing into them surfaced
two very different problems.

ConfigMgr Console installs cleanly with a bare `ConsoleSetup.exe /q`
on MECM01 itself, because MECM01 already has local site-server
registry context to fall back on; anywhere else, that context doesn't
exist and the installer needs to be told explicitly. Three attempts
came before the working syntax. Bare `/q` failed with
`DEFAULTSITESERVERNAME cannot be empty for silent or basic UI`, at
least naming the missing parameter in its own log. `/q
SiteServer=MECM01...` failed with `Invalid argument`, the wrong
parameter name entirely, worth reading the log more carefully before
guessing at the next attempt. `/q
DEFAULTSITESERVERNAME=MECM01...` alone failed with exit code 6,
`TARGETDIR cannot be empty`: a silent install needs an explicit
install path too, not just a site server. Only `/q
DEFAULTSITESERVERNAME=MECM01.myhomelab.hv.lab
TARGETDIR="C:\Program Files (x86)\Microsoft Configuration Manager"`
actually succeeded, exit code 0.

Windows Terminal is a UWP/MSIX app, preinstalled on both session
hosts, with no plain `.exe` to point a RemoteApp definition at. The
standard workaround publishes `explorer.exe` as the target with a
required command line that hands off to the app's AUMID:
`-CommandLineSetting Require -RequiredCommandLine
'shell:AppsFolder\Microsoft.WindowsTerminal_8wekyb3d8bbwe!App'`. The
AUMID came from Microsoft's documented well-known value rather than a
local lookup; `Get-StartApps` returned nothing over a PowerShell
Direct session on either session host, most likely because that
context has no interactive Start Menu cache to query. Worth flagging
honestly: this hadn't been click-tested end to end through a real RDP
or Web Access session at the time, only confirmed as correctly
registered and visible.

All four apps, ConfigMgr Console and Windows Terminal on
`RDCOL-Admin`, Windows Terminal and Notepad++ on `RDCOL-General`,
confirmed via `Get-RDRemoteApp` with `ShowInWebAccess = True`.
`New-RDRemoteApp` and `Get-RDRemoteApp` both have to run against
RDSCB, the actual broker, not locally on the Hyper-V host; the RD
cmdlets error out otherwise, which is easy to forget coming from other
role families where local admin tooling usually works from anywhere
domain-joined.

## What a user actually sees

The RD Web Access front end at `https://rdsgw.myhomelab.hv.lab/RDWeb`
is the part that matters most to anyone other than the person who
built it, so it's worth showing directly rather than just asserting it
works.

![RD Web Access login page, certificate trusted, no browser warnings]({{ '/assets/img/gallery/rds-web-access-login.png' | relative_url }})
_Signing in from WKS01: the `WebServerManualSAN` certificate resolves cleanly against `rdsgw.myhomelab.hv.lab`, no trust warnings_

![All four published RemoteApps listed after login]({{ '/assets/img/gallery/rds-web-access-remoteapps.png' | relative_url }})
_ConfigMgr Console and both Windows Terminal entries, Notepad++ alongside them: everything `Get-RDRemoteApp` reported, visible and clickable_

That confirms the web front end, the certificate, and the publishing
pipeline all the way through. It's still not the same claim as "the
Windows Terminal RemoteApp actually launches a working terminal
session," which is exactly the piece flagged above as not yet
click-tested; the icon showing up correctly here is necessary for that
to work, not sufficient proof that it does.

## Two Windows Terminal icons, and why Domain Admins was the wrong gate

The screenshot above shows something worth stopping on: two separate
Windows Terminal entries in the same resource list. That's a real bug,
not a display quirk, and it's worth walking through both because the
cause is easy to reproduce by accident and the fix generalizes well
past RDS.

Each collection restricts who can see it through a `UserGroup`
setting, and the original values were `RDCOL-Admin` gated on
`MYHOMELAB\Domain Admins`, `RDCOL-General` gated on `MYHOMELAB\Domain
Users`. Those two look like a reasonable admin/everyone split and
aren't one, because every domain account's primary group is Domain
Users by default, including every Domain Admins member. Anyone in
Domain Admins satisfies both filters at once, sees both collections'
resource lists merged in RD Web Access, and since Windows Terminal is
published separately in each collection under its own alias
(`WindowsTerminal-Admin`, `WindowsTerminal-General`), that user sees
it twice. Gating access on a privilege group like Domain Admins was
the actual mistake: it answers "does this account have elevated
rights," not "should this account see the admin tools," and the two
questions only look the same until someone new joins Domain Admins for
an unrelated reason and picks up RDS access as a side effect.

The first fix considered was a third collection, spanning both session
hosts, publishing Windows Terminal exactly once instead of duplicating
it. That doesn't work in this deployment model:

```
New-RDSessionCollection -CollectionName RDCOL-Common `
    -SessionHost RDSSH01.myhomelab.hv.lab, RDSSH02.myhomelab.hv.lab

WARNING: The RD Session Host server RDSSH01.myhomelab.hv.lab already exists in another collection.
WARNING: The RD Session Host server RDSSH02.myhomelab.hv.lab already exists in another collection.
Write-Error: Unable to create the session collection.
```

A session host can only belong to one session collection at a time; a
shared collection would mean pulling RDSSH01 and RDSSH02 out of
`RDCOL-Admin` and `RDCOL-General` entirely, breaking ConfigMgr Console
and Notepad++ publishing to do it.

The actual fix didn't need a third collection. Two new, disjoint AD
groups, `RDS-Admin-Users` and `RDS-General-Users`, living under
`OU=Groups,OU=LabOU`, replaced the privilege-group gate:
`RDS-Admin-Users` holds just `Administrator`; `RDS-General-Users`
holds `Administrator` and `test.labuser`, matching exactly who had
access under the old scheme, minus the accidental overlap. Repointing
each collection at its own group was the entire fix:

```powershell
Set-RDSessionCollectionConfiguration -CollectionName RDCOL-Admin -UserGroup 'MYHOMELAB\RDS-Admin-Users'
Set-RDSessionCollectionConfiguration -CollectionName RDCOL-General -UserGroup 'MYHOMELAB\RDS-General-Users'
```

Windows Terminal still gets published twice under the hood, unavoidable
given the one-collection-per-host constraint, but it's no longer
visible twice to anyone. `RDS-Admin-Users` and `RDS-General-Users`
don't overlap at all: `Administrator` sees ConfigMgr Console and its
own Windows Terminal icon from `RDCOL-Admin`, `test.labuser` sees
Notepad++ and its own Windows Terminal icon from `RDCOL-General`.
Neither account's resource list merges both collections anymore.

![Administrator's resource list after removing the group overlap entirely]({{ '/assets/img/gallery/rds-web-access-administrator-fixed-groups.png' | relative_url }})
_ConfigMgr Console and exactly one Windows Terminal icon, no Notepad++, no duplicate: `Administrator` now sees only what `RDS-Admin-Users` grants_

## Smart card sign-in, added without breaking the password path

Short version, before the detail: RD Web Access now accepts a YubiKey
PIV smart card, confirmed with an actual physical sign-in from WKS01,
and password sign-in still works too, just through the browser's
native credential prompt instead of the old branded page. Getting
there took four real pieces, TLS negotiation, an IIS module that
wasn't actually reaching HTTP.SYS, a platform constraint that forced a
full authentication-mode switch, and a loopback-testing trap, each one
covered below in the order they were actually found.

The original plan called for a second authentication option on RD Web
Access: YubiKey PIV smart card, sitting alongside the existing
password form rather than replacing it. The groundwork for this
already existed from an earlier project, the `SmartcardLogon`
certificate template, published on `myhomelab-DC01-CA` and confirmed
trusted domain-wide (the CA itself sits in the enterprise `NTAuth`
store, a prerequisite for any certificate it issues to be usable for
logon at all), plus a `SmartcardLogon` certificate already issued to
`test.labuser` from that earlier work. Nothing needed re-enrolling; RD
Web Access just needed to be told to accept a certificate as an
alternative to a password.

The IIS side of it came first, and the actual mechanics were less
obvious than "flip a setting in IIS Manager." RD Web Access already
had the underlying IIS role service installed,
`Client Certificate Mapping Authentication`, the AD-integrated module
that maps a presented certificate to a domain account automatically by
UPN, no manual per-user mapping required. It just wasn't turned on for
the site. Enabling it, and separately telling IIS to request a client
certificate during the TLS handshake, both hit the same wall on the
first attempt: `This configuration section cannot be used at this
path... locked at a parent level`. Both sections, `security/access`
(where the client-certificate SSL flag lives) and
`security/authentication/clientCertificateMappingAuthentication`, are
locked by default at the server level and have to be explicitly
unlocked with `appcmd unlock config` before a site can override them
at all.

Once unlocked, enabling the certificate-mapping module was
straightforward. Getting the TLS layer to actually ask for a
certificate was not. Setting the negotiate-client-certificate flag on
the `/RDWeb` application path specifically had no effect at all on the
real handshake, because that decision gets made at the TLS layer
before the server even knows which URL path was requested; it has to
be set at the site root instead. Even after correcting that and
restarting IIS, `netsh http show sslcert` still reported `Negotiate
Client Certificate: Disabled` on the actual HTTP.SYS binding, the
layer that TLS negotiation genuinely runs through. IIS's own
site-level setting had taken and persisted, it just wasn't being
pushed down to HTTP.SYS automatically the way older IIS documentation
describes. The fix was to update the HTTP.SYS binding directly:

```powershell
netsh http update sslcert ipport=0.0.0.0:443 `
    certhash=159f224d1848b420bbd563ed556b32a862851cbe `
    appid='{4dc3e181-e14b-4a21-b022-59fc669b0914}' `
    clientcertnegotiation=enable
```

That's the certificate hash and application ID already bound to port
443 on RDSGW, pulled straight from `netsh http show sslcert` rather
than guessed at; reusing the existing binding's own identifiers is
what makes this an update to the current binding instead of a
conflicting second one.

Rather than trust the configuration alone, the actual TLS handshake
got tested directly, from outside IIS entirely, using a raw
`SslStream` connection with a certificate-selection callback: the
callback only fires if the server has genuinely sent a
`CertificateRequest` during the handshake. It fired. The same test
also confirmed the handshake still completes cleanly with no
certificate supplied at all, proof the password path survived
untouched; `SslNegotiateCert` asks for a certificate, it doesn't
demand one.

That was still only the transport layer, and the first real test from
WKS01 exposed it: the browser correctly prompted to select a
certificate, but after picking it, RD Web Access redirected straight
back to the ordinary username and password page, as if nothing had
been presented at all. IIS's own logs made the gap obvious once
checked, `cs-username` was blank on every single request in that
window, meaning IIS had negotiated the certificate but never actually
turned it into an authenticated identity.

The real cause lived one layer up, in RD Web Access's own
`Pages\web.config`, and Microsoft has documented it directly inside
the file's own comments:

```
To turn on Windows Authentication:
    - uncomment <authentication mode="Windows"/> section
    - and comment out <authentication mode="Forms"> section
```

Forms and Windows authentication aren't two options a single RD Web
Access site can offer side by side; it's one or the other for the
whole site. Everything done so far, TLS negotiation, the certificate
mapping module, was necessary groundwork, but the application itself
was still hardcoded to its own branded password form and had no code
path that ever looked at the identity IIS had already established.
"Password stays default, smart card sits alongside it" was the
original framing, and it doesn't survive contact with how RD Web
Access is actually built.

Switching modes meant three changes to that one file, exactly as the
comment specifies: uncomment the `Windows` authentication line and
comment out the `Forms` block, comment out the `<modules>` section
that swaps in RD Web Access's own custom forms-cookie module, and
comment out a `<security>` block further down that was explicitly
forcing `anonymousAuthentication` on and `windowsAuthentication` off
for this specific path, overriding anything set elsewhere. On the IIS
side, that meant disabling Anonymous Authentication and enabling
Windows Authentication explicitly on `/RDWeb/Pages`, alongside the
certificate mapping module already turned on, then restarting IIS.

Confirming it actually worked needed two separate tests, because the
first one gave a misleading answer. Testing from RDSGW against its own
hostname returned a flat 401 no matter what credentials were supplied,
which looked like a broken configuration and wasn't: it's Windows'
loopback protection, a deliberate block on NTLM authentication against
a name that resolves back to the same machine, there specifically to
stop a reflection attack. Repeating the same test from a different VM
told the real story. The IIS log for that
request shows the full picture: an initial `401` challenge, a second
`401` to complete the NTLM handshake, and then a `302` carrying
`cs-username: MYHOMELAB\Administrator`, a fully authenticated,
successful sign-in. Anonymous access truly was gone; a password still
worked, just via the browser's native credential prompt instead of the
old styled page.

The actual test that mattered came last: `test.labuser`, signed into
WKS01, browsing to RD Web Access with the physical YubiKey inserted.

![Browser's native certificate picker offering test.labuser's YubiKey PIV certificate]({{ '/assets/img/gallery/rds-web-access-certificate-picker.png' | relative_url }})
_No RD Web Access page rendered yet, no password prompt either; this is the browser's own TLS-layer certificate picker_

![Signed in via smart card, landed directly on the correct two apps]({{ '/assets/img/gallery/rds-web-access-smartcard-signin-success.png' | relative_url }})
_Notepad++ and Windows Terminal, exactly what `RDS-General-Users` grants test.labuser, one Terminal icon, no password screen anywhere in the flow_

Signed straight in, no password screen, exactly the two RemoteApps
`RDS-General-Users` grants, confirming the earlier group-access fix
holds for a real smart-card session too, not just a password one.

## Lessons learned

- **Document what a build assumes already exists before documenting
  what it actually builds.** This post leans on three pieces of
  earlier work, the CA from the Phase 2 PKI build, the YubiKey PIV
  project's certificate template, the existing SQL AG, without
  building any of them fresh; naming that up front is what keeps the
  genuinely new work (five RDS roles, a certificate template, the
  group structure, smart card wiring) legible as new. Write the
  prerequisites down before the steps, not as an afterthought once
  someone asks what an unexplained reference to earlier work actually
  means.
- **Never gate an RD Session Collection's access on a built-in group
  like Domain Admins or Domain Users; design purpose-built groups for
  the intended audience.** A privilege group and an
  intended-audience group answer two different questions: Domain
  Admins answers "does this account have elevated rights," not "should
  this account see this specific set of RemoteApps," and it overlaps
  with Domain Users by default since every account's primary group is
  Domain Users regardless of whatever else it belongs to. Design
  dedicated groups per audience before wiring up `UserGroup`, not a
  built-in group even if it happens to include the right users.
- **A feature described as "add smart card sign-in alongside the
  password form" should be verified against how the actual product
  works before a plan is built on top of it.** RD Web Access only
  supports one authentication mode at a time for the whole site; Forms
  and Windows
  are a toggle, not a pair. The original plan assumed the two would
  coexist on the same branded page, and that assumption held right up
  until the site's own web.config said otherwise. Check what the
  platform actually supports before committing a plan to it, not after
  building most of the way there.
- **When an authentication test fails on the very first attempt from
  the server to itself, suspect the test before the configuration.**
  Windows blocks NTLM authentication against a hostname that resolves
  back to the same machine by default, a deliberate anti-reflection
  measure, not a sign anything is broken. Run the same test again from
  a different machine before trusting a same-machine failure; a local
  loopback test and a real cross-machine test are not interchangeable
  evidence.
- **A source-files error can come from a wrong assumption about the
  source, not a broken environment, and the same error against
  different sources is a signal, not a coincidence.** `sources\sxs`
  looks like the obvious install-media path, and for anything past a
  handful of legacy components, it isn't one; the real payload lives
  inside `install.wim`. Windows Update and local media failed
  identically here for that same reason, a bad path, not a bad source.
  Before blaming the network or the media, confirm the path actually
  contains what you're asking Windows to install.
- **A setting that saved successfully in IIS isn't guaranteed to reach
  the layer that actually enforces it.** The site-level
  negotiate-client-certificate flag persisted correctly in IIS, yet
  the real HTTP.SYS binding still reported it disabled. When a setting
  exists at both a config layer and a lower enforcement layer, check
  the enforcement layer directly rather than trust they stay in sync.

## Division of labor

The agent: every deployment and role-install command, isolating the
`sources\sxs` root cause after the two wrong theories, the certificate
template build, both RemoteApp collections including the ConfigMgr
Console argument troubleshooting and the Windows Terminal AUMID
workaround, tracing the duplicate Windows Terminal icon to the Domain
Admins/Domain Users overlap and building the replacement group
structure, and the full smart
card sign-in path, tracking the negotiate-client-certificate setting
down to the HTTP.SYS binding, finding the Forms/Windows exclusivity in
RD Web Access's own web.config, and the scripted cross-machine test
that confirmed password sign-in still worked. Me: the original farm
design (collection naming, the two SAN names, Per User over Per
Device), the standing call that broker HA and FSLogix each wait for an
explicit go-ahead before touching the SQL AG again, spotting the
duplicated Windows Terminal icon in the first place and asking for a
real group structure rather than a cosmetic fix, the go-ahead to
configure smart card sign-in once the earlier YubiKey groundwork was
in place, and the actual physical sign-in test from WKS01 that closed
the loop.

## What's next

The RDS farm is ready, publishes four RemoteApps, and now accepts
either a password or a YubiKey PIV smart card. Two pieces of the
original plan are still ahead, each one deliberately skipped rather
than folded into this base build: Connection Broker HA against the
SQL AG (database `RDS_CB`, its own two-step add-to-AG process, the
same pattern already used for SUSDB and the MECM site database)
matters because right now RDSCB is a single point of failure for the
whole farm; one broker outage takes every collection with it. FSLogix
profile containers on a new dedicated FS01 share matter because
without them, a user's profile is pinned to whichever session host
they happen to land on, which defeats the point of load-balancing
across two of them. NPS01 becoming Gateway's central policy store is a
parallel, non-blocking track; at this moment RDSGW works fine on local
policy until the NPS01 box exists.

Right behind this build came an experimental one on DEVOPS01: standing
up Microsoft `code serve-web` so VS Code runs in a browser instead of
on the desktop, which is where the `WebServerManualSAN` certificate
template built here gets reused for the first time.
