---
title: Homelab Build-Out — WSUS Sync, Patch Rings, and the SQL AG's Missing Log Backups
author: uzrg
date: 2026-08-05 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Microsoft, HyperV, Windows Server]
tags: [Microsoft, WSUS, ConfigMgr, MECM, SCCM, SQL Server, Availability Groups, pfSense, AI Agent]
pin: false
mermaid: false
---

# One WSUS sync failure, three fixes

WSUS01 had been failing every sync attempt with an error that looked like a
DNS problem, but as it turned out, it was not DNS at all. Getting WSUS to
sync was the first priority, then real patch management, followed by
trimming its disk footprint. While solving those three, the work also
uncovered a dead IPv6 gateway affecting the whole lab, a 37 GB SQL
transaction log that could never shrink, and a SQL Agent service that had
never run on any of the three Always On nodes.

**Bottom line:** WSUS syncs now, scoped to the three products actually
running in this lab (`Windows Server, version 1903 and later`, `Windows
11`, and `Microsoft SQL Server 2022`). Every domain-joined machine has a
real patch path through MECM, built as nine staggered rings so the domain
controllers and SQL AG nodes are never patched at the same time; the
guarded tier needs an explicit monthly go-ahead rather than running
unattended. And the SQL AG, which has been running with zero working
backup jobs, finally has one.

## The error that wasn't about DNS

The failure surfaced as a `WebException: The remote name could not be
resolved` for `sws.update.microsoft.com`, thrown from the sync client's
authentication call, even though `nslookup` and `Test-NetConnection`
both returned clean results. Sync history told a fuller story: several
failure modes appeared, not just DNS, including one timeout to a
completely different IP for the same hostname. That pattern pointed
toward a routing problem, not a DNS one.

pfSense's gateway status confirmed the diagnosis: `WAN_DHCP6`, the IPv6
WAN gateway, was down with 100% packet loss while IPv4 stayed fully
healthy. WSUS01 still had IPv6 enabled, and Microsoft's sync endpoint
resolves an AAAA record, so the client kept intermittently routing over a
gateway that no longer existed. pfSense stays read-only at all times, so
the real fix, setting WAN's IPv6 configuration to None, went through as a
separate, explicitly authorized change; disabling IPv6 on WSUS01's own
adapter unblocked the first sync in the meantime. The benefit reached
well beyond WSUS01: MECM01, untouched by any of this work, now falls back
to IPv4 automatically too, since pfSense no longer delegates a routable
IPv6 prefix to the LAN. One infrastructure fix resolved an issue
affecting the entire lab.

With connectivity resolved, one final step remained: WSUS01 had its own
patches pending a reboot mid-sync. The sync was stopped cleanly, a
checkpoint taken, the system rebooted, pending-reboot flags confirmed
clear, and the sync resumed. The resulting history reads exactly as
expected: failed, canceled during the reboot, then succeeded.

## Nine patch rings, six of them switched off by default

Once WSUS was healthy, the next requirement was real patch management:
every domain-joined machine covered, staggered so the domain controllers
and SQL AG nodes never go down together. That collides directly with a
standing guardrail prohibiting any state change on the domain
controllers, SQL AG nodes, or MECM servers without explicit confirmation
first. A single Automatic Deployment Rule firing every month cannot
satisfy that guardrail on its own.

The design resolves both at once: nine ring collections, each with its
own deployment off a single ADR, and maintenance windows staggered one
day apart:

| Ring | Systems | Window |
|---|---|---|
| 1 Pilot | WKS01 | 2nd Wed 20:00-22:00 |
| 2 General Servers | FS01, WSUS01, DHCP01 | 2nd Fri 01:00-03:00 |
| 3 MECM Passive | MECM02 | 3rd Mon 01:00-03:00 |
| 4 MECM Active | MECM01 | 3rd Tue 01:00-03:00 |
| 5 SQL Secondary | SQL02 | 3rd Wed 01:00-03:00 |
| 6 SQL Secondary | SQL03 | 3rd Thu 01:00-03:00 |
| 7 SQL Primary | SQL01 | 3rd Fri 01:00-03:00 |
| 8 Domain Controller | DC01 | 3rd Sat 01:00-03:00 |
| 9 Domain Controller | DC02 | 3rd Sun 01:00-03:00 |

![Patch ring collections in the MECM console]({{ '/assets/img/gallery/mecm-patch-rings-collections-general.png' | relative_url }})
_Rings 2-7 as real collections: General Servers, MECM Passive/Active, SQL AG Secondary/Secondary/Primary_

The domain controller rings are tracked separately, under their own
collection scope rather than the flat naming used above:

![Domain controller patch rings in their own collection scope]({{ '/assets/img/gallery/mecm-patch-rings-collections-dc-tier.png' | relative_url }})
_DC01 and DC02, each its own ring, kept apart from the general-purpose collection tree_

Rings 1 and 2 run unattended every cycle, since neither touches a guarded
system. Rings 3 through 9 exist fully configured, but with their
deployment flag set to `False`; that's the actual confirmation gate
built into the design, not a manual process layered on afterward. Every
guarded-tier ring is ready to go, but has to be switched on by hand, only
after the previous ring completes cleanly. For the SQL rings, that means
confirming Always On sync health first; live AG state (SQL01 primary,
SQL02 and SQL03 healthy secondaries) is why the secondaries patch first
and the primary goes last.

![The Automatic Deployment Rule's Deployment Settings, all nine collections listed]({{ '/assets/img/gallery/mecm-patch-rings-adr.png' | relative_url }})
_One ADR, nine deployments: Ring 1 (OP - All Workstations) and Ring 2 read Yes under Enabled, everything else reads No_

![Maintenance window on the SQL AG primary ring]({{ '/assets/img/gallery/mecm-patch-rings-maintenance-window-sql01.png' | relative_url }})
_Ring 7 (SQL01, SQL AG Primary): the actual window, third Friday, matching the table above_

Getting a working MECM console in place required its own detour:
installing it locally on MECM01 repeatedly failed with a generic "invalid
parameter" error and no log file, three unrelated cmdlet-syntax problems
layered together. `New-CMSchedule` requires `-DayOfWeek` and `-WeekOrder`
together for a monthly-by-weekday pattern, not the `-RecurInterval`
value, which doesn't exist for that case. `New-CMMaintenanceWindow
-ApplyTo` only accepts `SoftwareUpdatesOnly`, not the more
intuitive-looking alternative. And a deployment package created inline
via `-DeploymentPackageName` plus `-Location` fails outright regardless
of path type, while creating it as its own object first works without
issue. None of it surfaced as a distinct error message, so each had to be
isolated independently.

One smaller, cosmetic installer issue: the 32-bit OSD boot-image
extension fails to extract, returning `FDICopy failed with error code
11`, unrelated to corruption since Windows' own `expand.exe` extracts the
same cab cleanly. This is a known, benign failure in the ConfigMgr
console installer whenever the 32-bit boot image WIM isn't present, and
safe to ignore. The core console and PowerShell module install and
register fine before that step runs and aren't rolled back when it
fails, so the console works for everything patch-management related;
only OSD tooling is missing, which the lab doesn't need yet.

## The disk space problem that wasn't about WSUS

WSUS had been scoped to sync the entire "Windows" product family: 377
products spanning every Windows Server release back to 2003, every
client version, drivers, and language packs. Narrowing that to what's
actually running meant surveying OS version across every live VM
directly rather than guessing, which caught a real trap: a WSUS category
matching "Server 2025" turned out to be an Azure File Sync product name,
not the OS. The same shortcut on SQL Server would have gotten the version
wrong too; `@@VERSION` on SQL01 confirmed 2022, not the 2025 the name
suggested. The real scope: `Windows Server, version 1903 and later` (the
unified name every WS2019, WS2022, and WS2025 update publishes under),
`Windows 11`, and `Microsoft SQL Server 2022`.

That was the original ask. The investigation found something different:
WSUS's own content directory was never the problem, using only 0.13 GB,
since MECM stores update content separately in its own deployment
package. The real disk consumer was the SUSDB database itself, which
lives on the SQL Availability Group listener rather than locally on
WSUS01: 6.5 GB of actual data sitting next to a 37.4 GB transaction log
that had never once been backed up. SQL Server's own diagnostics
confirmed why: the `log_reuse_wait_desc` field showed the log stuck
waiting specifically on `LOG_BACKUP`, direct evidence that no backup had
ever run against this database.

SUSDB's membership in the Availability Group, alongside the MECM site
database, ruled out the usual standalone-WSUS fix of switching to SIMPLE
recovery: Always On requires FULL recovery for any database it
replicates, since replication ships the log continuously. The fix had to
work within FULL recovery, which means log backups, exactly what should
have been happening all along and wasn't.

A log backup cleared the `LOG_BACKUP` wait but revealed a chain behind
it: `OLDEST_PAGE`, then `AVAILABILITY_REPLICA`, then `LOG_BACKUP` again,
normal for a log this bloated since each corrective step generates new
log that itself needs backing up. Backup, checkpoint, and shrink looped
until the wait cleared (one more pass), taking the log from 37,448 MB to
520 MB. AG health, checked before and after every step, stayed `HEALTHY`
on all three nodes throughout.

## No SQL Agent, no log backups

Fixing SUSDB's log once doesn't prevent it recurring, so the next step
was a real, permanent log backup job. Querying `msdb.dbo.sysjobs` turned
up exactly one job, the default system history-purge. Nothing was
backing up logs anywhere on the AG, and SUSDB wasn't alone: `CM_MHL`, the
MECM site database, carried the identical unaddressed `LOG_BACKUP` wait.
Root cause: `SQLSERVERAGENT` was stopped on all three nodes, despite
being configured to start automatically on each. This wasn't a
deliberate configuration choice; the service had apparently never been
started since the AG went live.

The job has to be AG-aware, since SQL Agent jobs are scoped per-instance
and don't follow the listener: it checks whether the local replica holds
primary, then loops through every FULL-recovery database in the AG and
backs each one up:

```sql
IF EXISTS (
    SELECT 1
    FROM sys.dm_hadr_availability_replica_states ars
    JOIN sys.availability_replicas ar ON ars.replica_id = ar.replica_id
    WHERE ar.replica_server_name = @@SERVERNAME AND ars.role_desc = 'PRIMARY'
)
BEGIN
    DECLARE @dbname sysname, @path nvarchar(500), @ts nvarchar(20), @msg nvarchar(500)
    SET @ts = REPLACE(REPLACE(REPLACE(CONVERT(varchar(19), GETDATE(), 120),'-',''),':',''),' ','_')

    DECLARE db_cursor CURSOR FOR
        SELECT db.name
        FROM sys.databases db
        JOIN sys.availability_databases_cluster adc ON db.group_database_id = adc.group_database_id
        WHERE db.recovery_model_desc = 'FULL' AND db.state_desc = 'ONLINE'

    OPEN db_cursor
    FETCH NEXT FROM db_cursor INTO @dbname
    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @path = '\\FS01\SQLBackup\LogBackups\' + @dbname + '\' + @dbname + '_LOG_' + @ts + '.trn'
        SET @msg = 'Backing up log for ' + @dbname + ' to ' + @path
        RAISERROR(@msg, 0, 1) WITH NOWAIT
        BACKUP LOG @dbname TO DISK = @path WITH COMPRESSION
        FETCH NEXT FROM db_cursor INTO @dbname
    END
    CLOSE db_cursor
    DEALLOCATE db_cursor
END
ELSE
BEGIN
    PRINT 'Not primary replica -- skipping log backups on this node.'
END
```

Running every 15 minutes and deployed identically to all three nodes is
what makes it failover-safe without extra logic: whichever node holds
primary runs the backup, the other two fail the role check and exit as a
no-op. Starting the Agent service followed the AG's standard
one-at-a-time pattern: SQL01, then SQL02, then SQL03, health confirmed
`HEALTHY` before and after each. A manual test run before trusting the
schedule produced real backup files for SUSDB, CM_MHL, and AGSeed on the
first pass; final check showed `log_reuse_wait_desc` clear on SUSDB and
AGSeed, mid-transition on CM_MHL (expected), and the AG still `HEALTHY`
throughout.

## Lessons learned

- **A passing diagnostic only proves the diagnostic passed, not that the
  real workload will succeed.** `nslookup` and `Test-NetConnection` both
  succeeded while the sync client kept failing: the actual gap was which
  IP family was used on each attempt, something neither basic test
  checks. When a clean report contradicts a reported failure, test with
  the same protocol and path the real traffic uses for an accurate
  picture.
- **A "minor" protocol-level failure rarely stays contained to one
  server.** WSUS was the one system under investigation, but the
  underlying IPv6 gateway was dead lab-wide; fixing it changed routing
  behavior on every machine in the environment, including several nobody
  had suspected had an issue.
- **Build the approval gate into the automation, not around it.** Nine
  ring deployments, six of them disabled by default, actually satisfy a
  policy like "confirm before touching production"; one rule covering
  everything, however carefully scheduled, would not. If a safety rule
  matters, make the tooling structurally incapable of skipping it.
- **A generic error with no log file usually means the cmdlet never
  actually ran.** `New-CMSchedule`, `New-CMMaintenanceWindow`, and the
  inline deployment-package creation each failed here with the identical
  unhelpful message, for three completely unrelated reasons; isolating
  each parameter individually was the only way to find the real cause.
  Don't over-read a vague error as a description of what went wrong;
  treat it as a signal to test your inputs one at a time.
- **A recovery model is more of an intention than a guarantee.** With
  SIMPLE recovery, log space is reclaimed automatically at each
  checkpoint, so the log can't grow unchecked, but you accept from the
  start that point-in-time recovery isn't available. FULL recovery
  reverses that trade-off: SQL Server retains every transaction until a
  backup job explicitly tells it to release them. If that backup job
  never runs, nothing ever does, which is precisely what happened in
  this case. The log keeps expanding while the setting continues to
  report FULL, quietly suggesting a safety net that was never actually
  put in place. That's what makes it more dangerous than SIMPLE with
  nothing configured: with SIMPLE, you at least know what you're giving
  up. With FULL and no backups, no one realizes the problem until they
  need to restore to a specific point in time, look for a backup chain,
  and find it was never there. Check that the backup job actually runs.
  Don't just trust what the setting implies.
- **"Set to start automatically" and "currently running" are two
  different facts, and only one of them matters.** Every job depending on
  SQL Agent had silently done nothing since the Availability Group went
  live, and nothing in the AG's own health monitoring ever flagged it,
  because the two systems watch completely different things. AG health
  (`HEALTHY`, `SYNCHRONIZED`, the dashboard) is the Database Engine
  reporting on its own replication: is the log shipping to secondaries
  in real time, is failover safe right now. SQL Agent is a separate
  Windows service that runs scheduled work, including the log backup
  job, and the AG has no dependency on it and no visibility into it. The
  AG can report `HEALTHY` all day while the one thing responsible for
  keeping the log backed up is dead, because checking on Agent was never
  part of its job. Check actual service state directly; don't infer it
  from a startup-type setting, and don't assume a healthy AG means the
  whole SQL tier is being looked after.

## Division of labor

The agent: tracing the WSUS failure to pfSense's IPv6 gateway, the ring
and maintenance-window design, the ADR's nine deployments, isolating the
console/cmdlet-syntax problems, the OS/SQL version survey, finding the
SUSDB log bloat and its root cause, the log backup job's AG-aware design,
and every checkpoint along the way. Me: the pfSense WAN IPv6 fix itself
(always read-only for the agent), confirming the guarded-tier design and
the recovery-model constraint, and the go-ahead for the log backup fix
and new SQL Agent job.

## What's next

The guarded-tier rings still need their first real monthly cycle to
prove the manual-enable pattern in practice. The full and differential
backups sitting in `msdb.dbo.backupset` turned out to have a real
source: every one of them lines up, within seconds, with a Hyper-V
checkpoint taken against SQL01 for unrelated reasons. Hyper-V's
production checkpoints invoke the SQL Server VSS Writer inside the
guest, which performs a full backup of every database as part of
building an application-consistent snapshot. That's a genuine backup,
but not a substitute for a scheduled one: it only happens when someone
checkpoints the box, so coverage is exactly as reliable as the last
unrelated maintenance task that triggered one, not something to plan a
restore strategy around. Beyond that: the RD session-based farm, then
Operations Manager, then Azure DevOps.
