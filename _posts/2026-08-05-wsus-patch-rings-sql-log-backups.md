---
title: Homelab Build-Out — WSUS Sync, Patch Rings, and the SQL AG's Missing Log Backups
author: uzrg
date: 2026-08-05 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Microsoft, HyperV, Windows Server]
tags: [Microsoft, WSUS, ConfigMgr, MECM, SCCM, SQL Server, Availability Groups, pfSense, AI Agent]
pin: false
mermaid: false
---

# A WSUS sync failure that turned into three separate fixes

WSUS01 had been sitting there failing every sync attempt with an error that
looked like a DNS problem but wasn't. I asked the agent to get it syncing,
then build real patch management on top of it, then optimize its disk
footprint. Each of those three asks turned up something underneath it that
wasn't part of the original ask at all: a dead IPv6 gateway affecting the
whole lab, a 37 GB SQL transaction log that could never shrink, and a SQL
Agent service that had never been running on any of the three Always On
nodes.

**Bottom line:** WSUS syncs cleanly now, scoped to only the three products
actually running in the lab. Every domain-joined machine has a real patch
path through MECM, built as nine staggered rings so the domain controllers
and the SQL AG nodes are never mid-patch at the same time, with the guarded
tier requiring an explicit monthly go-ahead rather than running unattended.
And the SQL AG, which had been running since mid-July with zero working
backup jobs, now has one.

## The error that wasn't about DNS

The failure was a `WebException: The remote name could not be resolved` for
`sws.update.microsoft.com`, thrown from deep inside the WSUS sync client's
authentication call. Every manual check the agent ran said the opposite:
`nslookup` resolved the name fine, `Test-NetConnection` connected on 443
without issue, WinHTTP proxy was set to direct access. Sync history showed a
mix of failure modes across attempts, not just the DNS one, including a
plain connection timeout to a completely different IP for the same
hostname, which is what pointed the investigation away from DNS and toward
routing.

pfSense's gateway status confirmed it: `WAN_DHCP6`, the IPv6 WAN gateway,
was down with 100% loss, while the IPv4 WAN was perfectly healthy. WSUS01
still had IPv6 enabled, and Microsoft's sync endpoint resolves an AAAA
record over Azure Traffic Manager, so the sync client kept intermittently
trying to route over a gateway that wasn't there. pfSense is read-only for
the agent, always, so the actual infrastructure fix had to come from me:
setting WAN's IPv6 configuration to None. The agent's own workaround,
disabling IPv6 on WSUS01's adapter, is what got that first sync through
while I made the call on the real fix.

The pfSense-side fix turned out to help more than just WSUS01. Checking a
machine the agent had never touched, MECM01, showed it now carries only a
link-local IPv6 address, no global unicast address, no IPv6 default route
at all: with WAN IPv6 set to None, pfSense stops delegating a routable
prefix to the LAN, so every machine in the lab now falls back to IPv4
automatically instead of hanging against a dead gateway. One fix, whole-lab
effect.

With connectivity sorted, the sync itself needed one more push: WSUS01 had
patches of its own pending a reboot mid-sync. The agent stopped the sync
cleanly, checkpointed, rebooted (the box went through two boots finishing
cumulative update installation, which is normal), verified no pending-reboot
flags remained, and resumed. Final sync history reads exactly like it
should: failed, canceled (the reboot), succeeded.

## Patch rings, because "confirm before touching the SQL AG" and "patch everything automatically" don't mix

Once WSUS was healthy I asked for real patch management: every domain-joined
machine covered, but staggered so the domain controllers and the SQL AG
nodes never go down together. That collides directly with a standing rule:
no state change on the domain controllers, the SQL AG nodes, or the MECM
servers without my confirmation first. An Automatic Deployment Rule that
just fires every month isn't compatible with that rule by itself.

The agent's design was to build the ADR so it satisfies both asks at once.
Nine ring collections, one deployment per ring off a single ADR, each with
its own maintenance window one day apart:

| Ring | Systems | Window |
|---|---|---|
| 1 Pilot | WKS01 | 2nd Wed 20:00-22:00 |
| 2 General Servers | FS01, WSUS01, DHCP01 | 2nd Fri 01:00-03:00 |
| 3 MECM Passive | MECM02 | 3rd Mon 01:00-03:00 |
| 4 MECM Active | MECM01 | 3rd Tue 01:00-03:00 |
| 5 SQL Secondary | SQL02 | 3rd Wed 01:00-03:00 |
| 6 SQL Secondary | SQL03 | 3rd Thu 01:00-03:00 |
| 7 SQL Primary | SQL01 | 3rd Fri 01:00-03:00 |
| 8 | DC01 | 3rd Sat 01:00-03:00 |
| 9 | DC02 | 3rd Sun 01:00-03:00 |

Rings 1 and 2 are enabled and run unattended every cycle: neither touches a
guarded system. Rings 3 through 9 are created with their deployment flag set
to `False`. That's the actual confirmation gate, not a separate manual
process bolted on afterward: every guarded-tier ring exists, fully
configured, waiting, and has to be switched on by hand each month after I
confirm the previous ring came through clean. For the SQL rings that means
checking Always On sync health between each node; the agent queried live AG
state before designing the ring order and confirmed SQL01 was primary,
SQL02 and SQL03 secondary, both healthy, which is why the secondaries patch
first and the primary goes last.

Getting a working MECM console to build any of this took its own detour.
Installing it locally on MECM01 kept failing with a generic "invalid
parameter" error and no log file at all, which turned out to be three
separate cmdlet-syntax problems layered on top of each other: `New-CMSchedule`
needs `-DayOfWeek` and `-WeekOrder` together for a monthly-by-weekday
pattern, not a `-RecurInterval` value that doesn't actually exist for that
case; `New-CMMaintenanceWindow -ApplyTo` only accepts `SoftwareUpdatesOnly`,
not the value that seemed obvious; and creating a new deployment package
inline via `-DeploymentPackageName` plus `-Location` fails outright,
regardless of local or UNC path, for a reason still unexplained; creating
the package as its own object first and referencing it by name afterward
works fine. None of that showed up in a single error message, so each one
took isolating separately.

The installer itself had a smaller, mostly cosmetic problem: its 32-bit OSD
boot-image extension fails to extract with `FDICopy failed with error code
11`, confirmed unrelated to file corruption since Windows' own `expand.exe`
extracts the identical cab cleanly. The core console and PowerShell module
install and register fine before that step runs and aren't rolled back when
it fails afterward, so the console works for everything patch-management
related; OSD tooling just isn't installed, which the lab doesn't need yet.

## "Optimize disk space" surfaces a 37 GB transaction log

WSUS was scoped to sync the entire "Windows" product family: 377 products,
every Windows Server release back to 2003, every Windows client version,
drivers, language packs. I asked the agent to narrow that to what's actually
running. It surveyed OS version across every live VM directly rather than
guess, which caught one thing worth catching: a WSUS category matching
"Server 2025" turned out to be an Azure File Sync product name, not the OS,
and a similar shortcut on SQL Server would have gotten the version wrong
too, so it queried `@@VERSION` on SQL01 directly and got 2022, not the 2025
the name match suggested. The real scope, matching installed reality
exactly, turned out to be three products: `Windows Server, version 1903 and
later` (the unified name every WS2019/2022/2025 update publishes under;
there's no separate "Windows Server 2025" category), `Windows 11`, and
`Microsoft SQL Server 2022`.

That was the ask. What it actually found was that WSUS's own content
directory was never the problem: it sits at 0.13 GB, because MECM downloads
update content into its own deployment package rather than WSUS's local
store. The real disk consumer was SUSDB itself, which lives on the SQL AG
listener, not locally on WSUS01: 6.5 GB of data next to a 37.4 GB
transaction log. `log_reuse_wait_desc` read `LOG_BACKUP`. No log backup had
ever been taken against it.

SUSDB being an actual member of the Availability Group, alongside the MECM
site database itself, ruled out the usual standalone-WSUS advice of
switching it to SIMPLE recovery: Always On requires FULL recovery for any
database it replicates, since replication happens by continuously shipping
the log. The fix had to work within FULL recovery, which means log backups,
which is what should have been happening the whole time and wasn't.

With my go-ahead, the agent took a log backup, which cleared the
`LOG_BACKUP` wait but revealed a chain of others behind it, `OLDEST_PAGE`
then `AVAILABILITY_REPLICA` then `LOG_BACKUP` again, which is normal for a
log this bloated: each corrective step generates a small amount of new log
that itself needs a backup cycle before the old backlog can actually
release. It looped backup, checkpoint, and shrink until the wait cleared,
which took one more pass, then shrank the log file from 37,448 MB to 520 MB.
Availability Group health, checked before and after every single step,
stayed `HEALTHY` on all three nodes the entire time.

## The SQL Agent job that couldn't exist because SQL Agent wasn't running

Fixing SUSDB's log once doesn't stop it from happening again, so I asked
the agent to set up the actual log backup job. Querying `msdb.dbo.sysjobs`
for existing jobs turned up exactly one, the default system history-purge
job. Nothing was backing up logs anywhere on the AG, and it wasn't just
SUSDB exposed to it: `CM_MHL`, the MECM site database, had the identical
`LOG_BACKUP` wait sitting there unaddressed. Chasing why led to the actual
root cause: `SQLSERVERAGENT` was stopped on all three nodes, despite being
set to start automatically on all three. Not a deliberate configuration,
just something that had apparently never been started since the AG went
live.

The job itself has to be AG-aware, since SQL Agent jobs are per-instance and
don't follow the listener: it checks whether the local replica currently
holds the primary role before doing anything, and if it does, loops every
database in the AG that's in FULL recovery and backs each one up:

```sql
IF EXISTS (
    SELECT 1 FROM sys.dm_hadr_availability_replica_states ars
    JOIN sys.availability_replicas ar ON ars.replica_id = ar.replica_id
    WHERE ar.replica_server_name = @@SERVERNAME AND ars.role_desc = 'PRIMARY'
)
BEGIN
    DECLARE db_cursor CURSOR FOR
        SELECT db.name FROM sys.databases db
        JOIN sys.availability_databases_cluster adc
            ON db.group_database_id = adc.group_database_id
        WHERE db.recovery_model_desc = 'FULL' AND db.state_desc = 'ONLINE'
    -- backs up each one to \\FS01\SQLBackup\LogBackups\<db>\
END
```

Running every 15 minutes, deployed identically to all three nodes, which is
what makes it failover-safe without any extra logic: whichever node holds
primary at any given moment is the one that actually runs the backup, and
the other two see the role check fail and exit as a no-op. Starting the
Agent service followed the same one-at-a-time pattern as everything else on
this AG: SQL01, checkpoint the health check, then SQL02, then SQL03, AG
health confirmed `HEALTHY` before and after each. The agent test-ran the job
manually before trusting the schedule, and real backup files landed for
SUSDB, CM_MHL, and AGSeed on the first run. Final check showed
`log_reuse_wait_desc` clear on SUSDB and AGSeed, mid-transition on CM_MHL
(expected, not concerning), and the AG still `HEALTHY` on all three nodes
throughout.

## Lessons learned

- **A diagnostic that succeeds doesn't mean the real traffic pattern does.**
  `nslookup` and a bare `Test-NetConnection` both looked clean while the
  actual sync client kept failing intermittently, because the difference
  was which IP family got used, not whether the name resolved or the port
  was reachable.
- **A dead gateway that "only" affects IPv6 doesn't stay contained.** WSUS
  was the symptom that got investigated, but the pfSense fix changed
  routing behavior for the whole lab, including machines nobody had
  touched.
- **"Confirm before touching the guarded systems" has to be built into the
  automation, not layered on top of it.** Creating nine ring deployments
  with six of them switched off by default is what actually satisfies that
  rule; a single ADR covering everything wouldn't have, no matter how
  carefully it was scheduled.
- **A cmdlet failing with a generic error and no log file usually means the
  command line never got that far.** Three unrelated syntax problems all
  produced the identical unhelpful message; isolating each one required
  testing them independently rather than trusting the error text.
- **FULL recovery model without log backups is worse than SIMPLE recovery
  with none.** It looks like point-in-time recovery is available because
  the setting says FULL, but without backups actually running, none of that
  protection is real, and the log grows without bound in the meantime.
- **A service set to start automatically isn't the same as a service that's
  running.** Every job creation and every scheduled task depending on SQL
  Agent had been silently doing nothing since the AG went live, and nothing
  about the AG's own health checks would have surfaced that on its own.

## Division of labor

The agent: every diagnostic step tracing the WSUS failure to pfSense's
IPv6 gateway, the ring collection and maintenance window design, the ADR
and its nine deployments, isolating the console install and cmdlet-syntax
problems, the OS and SQL version survey that caught the Server-2025 and
SQL-2022 naming traps, finding the SUSDB log bloat and its actual root
cause, the log backup job's AG-aware design, and every checkpoint and
health check along the way. Me: the pfSense WAN IPv6 fix itself, since
pfSense stays read-only for the agent always, confirming the guarded-tier
deployment design and the SUSDB recovery-model correction, and the
go-ahead for the log backup fix and the new SQL Agent job.

## What's next

The guarded-tier rings need their first real monthly cycle to prove the
manual-enable pattern in practice, not just in design. Full and differential
backups for the AG appear to be running through some mechanism that predates
this fix, never identified during this pass; worth confirming what it
actually is before assuming it's solid. Beyond that, the roadmap holds where
it's held for a while now: the RD Session-based farm, then Operations
Manager, then Azure DevOps.
