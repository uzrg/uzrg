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

The [last MECM post]({% post_url 2026-07-26-mecm-dp-checkpoint-reverts %}) left
the Yubico Smart Card Minidriver running as a legacy Package on four pilot
machines. This closes that out: convert it to a proper Application, then
deploy it lab-wide. It's Available on the domain controllers and Required
everywhere else.

**Bottom line:** the conversion itself was trivial. The rollout wasn't. Two
issues surfaced, neither where expected: three AD computer objects that
looked like domain members but were pre-staged reservations, and the
"DEP - Production - Yubico Minidriver" collection silently capped at one
machine by a bad limiting-collection scope. Both are fixed now and every
in-scope machine runs the Application; the legacy Package is retired.

## Converting the Package

The legacy Package (`MHL00006`) ran
`msiexec /i YubiKey-Minidriver-5.0.4.273-x64.msi /quiet /norestart` from
`\\FS01\MECMSource\YubicoMinidriver\`. The Application needed the MSI's
product code for its detection rule, pulled directly from the MSI's
`Property` table rather than guessed:

```powershell
# Point this at your own MSI - path below is this lab's source share, not a general default
$msiPath = "\\FS01.myhomelab.hv.lab\MECMSource\YubicoMinidriver\YubiKey-Minidriver-5.0.4.273-x64.msi"
$installer = New-Object -ComObject WindowsInstaller.Installer
$db = $installer.GetType().InvokeMember("OpenDatabase", "InvokeMethod", $null, $installer, @($msiPath, 0))
$view = $db.GetType().InvokeMember("OpenView", "InvokeMethod", $null, $db,
    @("SELECT Property, Value FROM Property WHERE Property = 'ProductCode'"))
$view.GetType().InvokeMember("Execute", "InvokeMethod", $null, $view, $null)
$record = $view.GetType().InvokeMember("Fetch", "InvokeMethod", $null, $view, $null)
$productCode = $record.GetType().InvokeMember("StringData", "GetProperty", $null, $record, 2)
$productCode
```

Product code: `{A8C8D1E1-3BB1-470D-BBD8-3AE20FB0FD85}`. One MSI deployment
type, same install/uninstall commands, MSI-product-code detection,
install-for-system. Content stayed on the same FS01 source; the earlier
content-library relocation never touched this path. Distributed cleanly to
MECM01; the legacy Package's deployment was retired once the Application went
live on the same collection.

## Scoping "domain-joined" correctly

Before deploying, every AD computer object was audited against Hyper-V's
actual running state. Easy exclusions: VMM01 (standing hold), and the five
RD-role VMs (Phase 3 hasn't started; left off rather than booted just for
this).

Three more looked like real targets (NPS01, DEVOPS01, OPSMGR01), each with a
proper AD object in the right OU. ConfigMgr's AD System Discovery had already
rejected all three. Cause: blank `operatingSystem`/`dNSHostName`,
`lastLogonTimestamp` still epoch, `pwdLastSet` unchanged since object
creation; one even still answered to `WIN-QI4D62II366`. These are pre-staged
reservations, not domain members, and out of scope, thus nine real targets
for Required (DHCP01, FS01, WSUS01, WKS01, MECM01, MECM02, SQL01-03) and
DC01/DC02 for Available.

## Rolling out the client

Four of nine already had the client. The rest, plus both DCs, got it via
[`configmgr/Install-SCCMClient.ps1`](https://github.com/uzrg/powershell-toolkit/blob/main/configmgr/Install-SCCMClient.ps1),
which resolves site code and management point from AD's System Management
container instead of hardcoding either.

Standard guardrails applied on every protected tier: checkpoint, install,
verify, next, one node at a time. SQL AG: secondaries first (SQL02, SQL03),
primary last (SQL01), sync health checked before and after each:

```sql
SELECT ar.replica_server_name, ars.role_desc, ars.synchronization_health_desc
FROM sys.dm_hadr_availability_replica_states ars
JOIN sys.availability_replicas ar ON ars.replica_id = ar.replica_id
```

All three stayed `HEALTHY` throughout. Same discipline applied to the DCs:
`repadmin /replsummary` and `dcdiag /q` clean before and after each, with
zero replication failures.

## Chasing a deployment that showed no progress

Client installed everywhere, both deployments live, but nothing happened!
`Get-CMApplicationDeploymentStatus` showed one machine reporting out of
eleven, and a console-triggered policy refresh didn't help. `PolicyAgent.log`
on SQL01 gave the real answer: *"No new assignments for Machine SQL01."*

That pointed at the collection, not the client. `SMS_FullCollectionMembership`,
queried directly over WMI, showed one actual member, WKS01, despite nine
direct membership rules. Root cause: the collection's limiting collection was
scoped to "OP - All Workstations," which excludes servers entirely. A
collection can never contain anything outside its limiting collection, so
every server target was silently dropped. The collection had sat empty since
being staged for a rollout that never shipped.

The actual fix is one collection property change plus a forced
re-evaluation, once the session is connected to the site:

```powershell
# Set-CMDeviceCollection and Invoke-CMCollectionUpdate only exist once the
# ConfigurationManager module is loaded and you're sitting in the site's
# PSDrive - swap MHL / MECM01 below for your own site code and site server
Import-Module "$($env:SMS_ADMIN_UI_PATH)\..\ConfigurationManager.psd1"
if (-not (Get-PSDrive -Name MHL -PSProvider CMSite -ErrorAction SilentlyContinue)) {
    New-PSDrive -Name MHL -PSProvider CMSite -Root MECM01.myhomelab.hv.lab -Description "MECM Site" | Out-Null
}
Set-Location "MHL:\"

Set-CMDeviceCollection -Name "DEP - Production - Yubico Minidriver" -LimitingCollectionName "All Systems"
Invoke-CMCollectionUpdate -Name "DEP - Production - Yubico Minidriver"
```

All nine members resolved within seconds. Triggering the policy and
evaluation cycles directly on each client (the console's push notification
still wasn't reliable) closed the loop in under a minute; SQL01 went from
"no new assignments" to a verified install in thirty seconds.

## Verifying the result

Console-level status stayed unreliable: it's driven by the site's periodic
status summarizer, not real-time client state. The real signal was each
client's own `AppDiscovery.log` and `AppEnforce.log`; MECM01 logs its own
components to `C:\Program Files\SMS_CCM\Logs` instead of the usual
`C:\Windows\CCM\Logs`, worth knowing before assuming a machine shows no
activity.

Final state, confirmed machine by machine: all nine Required targets show a
successful install or clean compliant detection, and both DCs correctly show
the Application as not installed, exactly the Available behavior specified.

## Lessons learned

- **A limiting collection is a hard ceiling, not a suggestion.** Direct
  membership rules only resolve for resources already inside the limiting
  collection: no error, no warning, just membership that silently never
  happens. Check the limiting collection first when a device won't join.
- **Aggregate deployment status lags reality.** The console and API views
  depend on the site's status summarizer cycle. For real-time truth, go to
  the client's own `AppDiscovery.log`/`AppEnforce.log`.
- **A pre-staged AD computer object isn't proof of domain membership.**
  Check `lastLogonTimestamp` and `pwdLastSet` before trusting an OU
  placement.
- **Don't rely on push notifications for delivery.** Trigger the policy or
  evaluation cycle locally on the client when timing matters.

## Division of labor

The agent: the Application build, content distribution, the domain-joined
audit, every checkpointed install, and root-causing the collection bug.
Me: the Available-versus-Required split, skipping the powered-off RD-role
VMs, and authorizing the domain-controller installs once the permission
layer balked.

## What's next

NPS01, DEVOPS01, and OPSMGR01 pick up this Application automatically once
each is built and joined; no collection change needed. The RD-role VMs get
it when Phase 3 starts. Next: the RD session-based farm, then Operations
Manager, Azure DevOps, and Linux.
