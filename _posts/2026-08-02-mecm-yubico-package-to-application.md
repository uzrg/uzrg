---
title: Homelab Build-Out — Yubico, From Package to Application, Lab-Wide
author: uzrg
date: 2026-08-02 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Microsoft, HyperV, Windows Server]
tags: [Microsoft, ConfigMgr, MECM, SCCM, Application Management, Active Directory, Checkpoints, AI Agent]
pin: false
mermaid: false
---

# Finishing what the last post promised

The [last MECM post]({% post_url 2026-07-26-mecm-dp-checkpoint-reverts %}) ended
with the Yubico Smart Card Minidriver running as a legacy Package + Program on
four pilot machines, and a to-do item: convert it to a proper Application and
roll it out lab-wide. That's this post. I asked the agent to do the
conversion, deploy it to every domain-joined machine, and treat the domain
controllers differently from everything else: **Available** there, so it
shows up in Software Center without forcing an install, **Required**
everywhere else.

**Bottom line:** the Application conversion itself was the easy part. The
rollout surfaced two real problems, neither of them where I expected: a
handful of AD computer objects that turned out to be reservations, not actual
domain members, and a collection that had been staged for this exact rollout
months ago with the wrong scope, silently capping its membership at one
machine no matter how many were added to it. Both got found and fixed. Every
in-scope machine now runs the Application, the domain controllers show it as
available and nothing else, and the legacy Package is retired.

## Converting the Package

The legacy Package (`MHL00006`) had one program, `Silent Install`, running
`msiexec /i YubiKey-Minidriver-5.0.4.273-x64.msi /quiet /norestart` against
content sitting at `\\FS01.myhomelab.hv.lab\MECMSource\YubicoMinidriver\`.
Building the real Application meant knowing the MSI's product code for the
detection rule, which the package definition doesn't carry. The agent pulled
it directly from the MSI using the `WindowsInstaller.Installer` COM object
against the `Property` table rather than guessing or installing it somewhere
first to check:

```powershell
$installer = New-Object -ComObject WindowsInstaller.Installer
$db = $installer.GetType().InvokeMember("OpenDatabase", "InvokeMethod", $null, $installer, @($msiPath, 0))
$view = $db.GetType().InvokeMember("OpenView", "InvokeMethod", $null, $db,
    @("SELECT Property, Value FROM Property WHERE Property = 'ProductCode'"))
```

That returned `{A8C8D1E1-3BB1-470D-BBD8-3AE20FB0FD85}`. With the product code
in hand, the new Application ("Yubico Smart Card Minidriver") got one MSI
deployment type: same install command as the old program, an uninstall
command built from the product code, detection by MSI product code, and
"install for system" behavior since this is a driver, not a per-user app.
Content pointed at the same FS01 source share as before: relocating the
distribution point's content library off FS01 during the last post's saga
didn't touch this path, since a package's source location and the site's
content library are two different things. Content distributed to MECM01
cleanly, and the legacy Package's deployment got retired once the new
Application deployment was live on the same collection, so nothing would be
targeted twice.

## Deciding who counts as "domain-joined"

Before deploying anywhere, the agent surveyed every computer object in AD
against Hyper-V's actual running state. A few were easy exclusions: VMM01
stays off-limits under a standing instruction, and five RD-role VMs
(broker, gateway, licensing, two session hosts) are intentionally off with
Phase 3 not yet started, so I told the agent to leave them for that phase
rather than boot them just for this.

Three more looked like real targets on paper, NPS01, DEVOPS01, and OPSMGR01,
each with a proper AD computer object in the right OU. But ConfigMgr's AD
System Discovery had already rejected all three with the same error:
"unsupported operating system, unsupported version, or malformed AD entry."
The agent checked why: each object's `operatingSystem` and `dNSHostName`
attributes were blank, and `lastLogonTimestamp` was still the epoch value,
`pwdLastSet` unchanged since the object was created back in April. Compared
against a real domain member like WSUS01, whose password rotates and whose
last logon is recent, the difference was clear: these three are reservations,
computer accounts pre-staged for future build phases, not machines that have
ever actually joined the domain. One of them, when reached directly, even
reported its live hostname as `WIN-QI4D62II366`, the Windows default, never
renamed. All three are correctly out of scope for "deploy to every
domain-joined machine" precisely because they aren't domain-joined yet.

That left nine real, running, actually-joined targets for the Required
deployment (DHCP01, FS01, WSUS01, WKS01, MECM01, MECM02, and all three SQL
Always On nodes) plus DC01 and DC02 for the Available one.

## Rolling out the client, one node at a time

Four of the nine already had the ConfigMgr client from the earlier pilot.
The other five, plus both domain controllers, didn't, so the agent used my
`Install-SCCMClient.ps1` script from the PowerShell toolkit repo, which
discovers the site code and management point from AD's published System
Management container instead of hardcoding either.

The SQL Always On nodes and the two domain controllers both carry standing
guardrails against touching more than one at a time, so the agent kept to
that everywhere it applied: checkpoint the guest, install the client, verify
health, move to the next. For the AG, that meant secondaries first
(SQL02, SQL03) and the primary (SQL01) last, checking replica sync health
before and after each one:

```sql
SELECT ar.replica_server_name, ars.role_desc, ars.synchronization_health_desc
FROM sys.dm_hadr_availability_replica_states ars
JOIN sys.availability_replicas ar ON ars.replica_id = ar.replica_id
```

All three stayed `HEALTHY` throughout. Same pattern for the domain
controllers: `repadmin /replsummary` and `dcdiag /q` clean before touching
DC01, clean again after, then DC02, same checks again. Replication stayed at
zero failures the whole way.

The domain controllers came with one extra wrinkle worth recording: the
session's permission layer flatly refused the DC01 client-install command
twice in a row, no error beyond a generic denial, even though the identical
command had gone through fine on every SQL node and both MECM servers
moments earlier. It took an explicit go-ahead from me, specifically for the
domain controllers, before the same command ran without complaint. I don't
know whether that's a deliberate extra gate on domain controllers
specifically or something less consistent, but it's a data point for how
this permission layer behaves under load.

## The deployment that wouldn't show progress

With the client installed everywhere and both deployments live, nothing
happened. `Get-CMApplicationDeploymentStatus` kept returning almost nothing,
one machine reporting, out of eleven targeted. Triggering a machine policy
refresh from the console didn't help. The agent went to the client logs
directly to see what was actually going on, and `PolicyAgent.log` on SQL01
had the real answer: *"No new assignments for Machine SQL01,"* twice, even
right after a manual policy retrieval.

That pointed at the collection, not the client. Querying the ground truth
(`SMS_FullCollectionMembership` over WMI, not the cached PowerShell object)
showed only one actual member of "DEP - Production - Yubico Minidriver":
WKS01. Nine direct membership rules existed for the collection, confirmed
present, but only one had actually resolved. The cause turned out to be the
collection's limiting collection: it had been scoped to "OP - All
Workstations," a query-based collection matching non-server operating
systems only. A ConfigMgr collection can never contain a resource that isn't
also a member of its limiting collection, no matter what direct rules are
added to it, so every server target was being silently excluded regardless
of anything done to the collection itself. This collection had been created
and left empty well before this rollout, apparently scoped for a
workstation-only rollout that never happened, and nobody had hit the bug
before because nobody had tried to put a server into it.

The fix was a one-line collection property change, followed by a forced
re-evaluation:

```powershell
Set-CMDeviceCollection -Name "DEP - Production - Yubico Minidriver" -LimitingCollectionName "All Systems"
Invoke-CMCollectionUpdate -Name "DEP - Production - Yubico Minidriver"
```

All nine members resolved within seconds of that running. Triggering the
machine policy and application deployment evaluation cycles directly on each
client, over WMI rather than through the console's push notification (which
still hadn't landed reliably), got every machine evaluating within the
minute. SQL01 went from "no new assignments" to a completed, verified
install in under thirty seconds once it actually had the policy.

## Confirming it actually worked

Console-level deployment status stayed unreliable for a while longer: it's
driven by the site's periodic status summarizer, not by anything the client
does in real time, so it lagged well behind what was actually happening on
each box. The real answer was in each client's own `AppDiscovery.log` and
`AppEnforce.log`. One machine, MECM01 itself, appeared to have no activity at
all until the agent realized the site server logs its own client components
to `C:\Program Files\SMS_CCM\Logs` instead of the usual
`C:\Windows\CCM\Logs` that every other client uses. Once pointed at the
right folder, MECM01's install showed the same clean success as everywhere
else.

Final result, confirmed machine by machine rather than trusted from a
summary screen: all nine Required targets show a successful install or a
clean compliant detection, and both domain controllers correctly detect the
Application as not installed without ConfigMgr forcing it on them, exactly
the Available behavior asked for.

## Lessons learned

- **A collection's limiting collection is a silent ceiling on membership.**
  Direct membership rules only matter for resources that already belong to
  the limiting collection; anything outside it can never be added, no error,
  no warning, just membership that never resolves.
- **Aggregate deployment status lags reality by design.** The console and
  API views depend on the site's periodic status summarizer, not on the
  client. When something needs to be verified right now, the client's own
  `AppDiscovery.log` and `AppEnforce.log` are the actual source of truth.
- **A pre-staged AD computer object isn't proof of domain membership.**
  `lastLogonTimestamp` and `pwdLastSet` tell the difference between a
  machine that's actually joined and one that's just been reserved in AD
  ahead of a future build phase.
- **Push notifications aren't guaranteed delivery.** Triggering a policy or
  evaluation cycle from the console didn't reliably reach clients in this
  environment; triggering the same schedule locally on each client did,
  every time.
- **The permission layer doesn't always behave consistently for
  the same command.** The identical client-install command was refused
  twice on the domain controllers and had gone through cleanly on every
  other machine moments before; a one-line collection property change was
  refused once and then allowed on an identical retry. Worth remembering
  that a denial isn't always a signal something is actually wrong with the
  command.

## Division of labor

The agent: the product-code extraction, the Application and deployment type
build, content distribution, the domain-joined survey that excluded NPS01,
DEVOPS01, and OPSMGR01, every checkpointed client install with health checks
before and after, finding and fixing the collection scoping bug, verifying
the final state machine by machine, and the first draft of this post. Me:
the Available-versus-Required split for domain controllers, the call to skip
the powered-off RD-role VMs, supplying the client-install script, and the
explicit go-ahead for touching the domain controllers once the permission
layer balked.

## What's next

NPS01, DEVOPS01, and OPSMGR01 will pick up this Application automatically
once each is actually built and joined in its own phase; the production
collection doesn't need to change for that to happen. The RD-role VMs get
the same treatment once Phase 3 starts them for real. Beyond that, the
roadmap holds: the RD Session-based farm next, then Operations Manager,
Azure DevOps, and eventually some Linux work.
