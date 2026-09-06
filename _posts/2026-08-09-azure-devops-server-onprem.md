---
title: Homelab Build-Out — Setting Up Azure DevOps Server On-Prem on a SQL AG
author: uzrg
date: 2026-08-09 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Microsoft, HyperV, Windows Server]
tags: [Microsoft, Azure DevOps, SQL Server, Availability Groups, IIS, Active Directory]
pin: false
mermaid: false
---

# Installing, configuring, and troubleshooting an on-prem Azure DevOps Server

This is a full build walkthrough for standing up Azure DevOps Server
on-premises with its databases on a SQL Server Always On Availability
Group, fronted by IIS with a proper certificate, and using Windows
Active Directory for sign-in, no separate DevOps password to manage.
Written granular enough that someone reasonably comfortable with Windows
Server, IIS, and basic SQL Server administration should be able to
reproduce the whole thing, hit the same rough edges this build hit, and
recognize them immediately instead of re-diagnosing from scratch.

**End state**: `https://devops.myhomelab.hv.lab/DefaultCollection/LabOps`
live, domain accounts signing in with zero prompts, databases living as
proper members of the existing SQL AG.

## What you need before starting

- A domain-joined Windows Server host for the application tier (this
  build used Windows Server 2025, **check the version table in Step 1
  before you assume your existing install media works**).
- A SQL Server instance or Availability Group listener the host can
  reach, with SQL Server Enterprise/Standard (not Express, for anything
  beyond personal use).
- An internal CA already issuing certificates to domain members.
- Local admin on the application-tier host, and `sysadmin` on the SQL
  side for whoever runs the install (temporarily; can be revoked after).
- Azure DevOps Server installation media matching your actual OS and SQL
  Server versions. See Step 1: this is the step most likely to trip you
  up if skipped.

## Step 1: Confirm your install media actually supports your OS and SQL version

Do this before downloading or attaching anything. Azure DevOps Server's
version support is a hard matrix, not a soft recommendation: an
unsupported combination can fail partway through setup in ways that are
hard to untangle after the fact.

Check Microsoft's current requirements page for exactly two things: the
**Server operating systems** table and the **SQL Server** support table.
As of this build:

| Azure DevOps Server version | Supported host OS | Supported SQL Server |
|---|---|---|
| Azure DevOps Server (current, unversioned) | Windows Server 2025, Windows Server 2022 | SQL Server 2025, SQL Server 2022 |
| Azure DevOps Server 2022.2 | Windows Server 2022, Windows Server 2019 | SQL Server 2022, SQL Server 2019 |

**This build's first mistake**: an older `azuredevops2022.2.iso` was
already sitting in the lab's media library and looked like the obvious
choice. It does not support Windows Server 2025. The fix wasn't
downgrading the server; it was going to Microsoft's actual download
page (`Server & Tools > Azure DevOps Server Download`) and pulling the
current, unversioned release instead, which explicitly lists Server 2025
as supported. If your existing media doesn't match your host OS in that
table, get different media before doing anything else. Don't install
anyway and hope.

## Step 2: Decide on service accounts (and don't assume gMSA works here)

You'll need at minimum:
- **An application-tier service account**, the identity IIS and the
  Azure DevOps services run as.
- **A build agent service account**, if you're planning to run pipeline
  agents on the same box later.

If your environment already uses group Managed Service Accounts (gMSA)
for SQL Server or other services, the instinct is to keep using them
here. **Check first.** Azure DevOps Server's own service-account
documentation only describes built-in accounts (`NT AUTHORITY\NETWORK
SERVICE`) or standard domain user accounts for the application-tier
service. There's no gMSA option documented anywhere, and it's a
long-standing, still-unanswered question in Microsoft's own developer
community. Absence of a "yes" in the docs isn't a "yes."

This build used two plain domain service accounts instead:
`svc-adoserver` (application tier) and `svc-adoagent` (reserved for a
future build agent). Both need "Log on as a service" rights, which
`tfsconfig unattend` grants automatically during configuration, so you
don't need to pre-stage that permission by hand.

The SQL Server engine itself doesn't need a new account if it's already
running under a gMSA from an earlier build. These new databases just
join the existing instance/AG under whatever account SQL Server already
runs as.

## Step 3: Prepare SQL Server: logins and the Full-Text Search feature

Two SQL-side prerequisites, both worth checking before you ever run
setup, not after it fails.

**3a. Grant SQL access to the accounts that need it.** The account
running setup needs a real login with meaningful rights (this build
granted `sysadmin`, temporarily, per Microsoft's own documented
guidance; you can scope it down or revoke it after setup completes).
The application-tier service account needs a login too, though its
actual database-level permissions get created automatically once its
databases exist.

```sql
CREATE LOGIN [DOMAIN\YourSetupAccount] FROM WINDOWS;
ALTER SERVER ROLE sysadmin ADD MEMBER [DOMAIN\YourSetupAccount];
CREATE LOGIN [DOMAIN\svc-adoserver] FROM WINDOWS;
```

If your SQL Server is an Availability Group, **run this identically on
every replica**, not just the primary. A login created only on today's
primary won't exist if the AG ever fails over.

**3b. Confirm Full-Text Search is actually installed.** Azure DevOps
Server requires it outright, and it's easy to have a SQL Server instance
that's missing it, and this build's AG had all three nodes without it,
despite SQL Server otherwise being fully functional for years.

```sql
SELECT SERVERPROPERTY('IsFullTextInstalled') AS FullTextInstalled;
```

If that returns `0` or blank, you need to add the feature via SQL Server
Setup. **If this is an Availability Group, don't add the feature to one
node and call it done.** Full-Text Search has to go on every replica
individually, one node at a time, with AG health verified healthy
between each, or you'll end up with a working primary and secondaries
that can't take over. The full rolling procedure, plus the exact
command and the wrong parameter name that burns ten minutes if you
guess at it, is in the Troubleshooting section below.

## Step 4: Certificate and DNS

Request (or reuse) a certificate covering your Azure DevOps Server's
hostname, and create the DNS record pointing at the application-tier
host. This build reused the same internal-CA template pattern from
earlier work, nothing Azure-DevOps-specific here.

**One hostname, used in five places, and they all have to match
exactly**: the certificate's CN, the DNS A record, and three separate
references in Step 6's unattended config (`UrlHostNameAlias`,
`PublicUrl`, and the hostname embedded in `SiteBindings`). Pick the
real hostname now and use the identical value everywhere below; a
mismatch in any one of these is what turns into a certificate warning
or a binding that silently doesn't match what a browser actually
requests.

**One thing worth checking now, not after setup fails**: if this host
already serves other HTTPS sites on port 443, confirm those existing
bindings use **SNI** (Server Name Indication), not a single non-SNI
certificate bound to `0.0.0.0:443`. HTTP.sys only allows one non-SNI
certificate per IP:port. Azure DevOps Server's setup will try to add
its own binding on 443 and fail outright if something else already
claimed that port without SNI. See the Troubleshooting section for the
exact fix if you hit this.

## Step 5: Install the binaries

Mount the ISO and run the installer silently:

```powershell
Start-Process -FilePath 'D:\AzureDevOps.exe' -ArgumentList '/Silent' -Wait
```

Confirm it actually landed before moving on:

```powershell
Get-ChildItem 'C:\Program Files\Azure DevOps Server'
```

You should see `Application Tier`, `Tools`, `Search`, and a few other
folders. `Tools\TfsConfig.exe` is what you'll use for every step from
here forward, no GUI wizard needed anywhere in this process.

## Step 6: Generate and fill in the unattended configuration file

```powershell
$tfsConfig = 'C:\Program Files\Azure DevOps Server\Tools\TfsConfig.exe'
& $tfsConfig unattend /create /type:NewServerBasic /unattendfile:'C:\Windows\Temp\ado-unattend.ini'
```

Open the generated `.ini` and change these values:

| Setting | Change to |
|---|---|
| `InstallSqlExpress` | `False` (unless you genuinely want SQL Express) |
| `SqlInstance` | Your AG listener name, or `Server\Instance` for a standalone SQL Server |
| `DatabaseLabel` | A short label, controls the actual database names, e.g. `LabOps` produces `AzureDevOps_LabOpsConfiguration` |
| `IsServiceAccountBuiltIn` | `False` |
| `ServiceAccountName` | `DOMAIN\svc-adoserver` |
| `UrlHostNameAlias` | Your real hostname, e.g. `devops.yourdomain.local` |
| `SiteBindings` | `https:*:443:yourhostname:My:<certificate thumbprint, no spaces>` |
| `PublicUrl` | `https://yourhostname/` |

**The one value that can't go through the normal edit path**: the
service account's password. It has to be added as a literal line inside
the `[Configuration]` section of the ini file itself:
`ServiceAccountPassword=<password>`, never via the `/inputs:` parameter
on the command line, which explicitly refuses to carry secret values.
Delete the ini file the moment configuration succeeds; it sits there in
plaintext until you do.

## Step 7: Verify, then configure

Always run `/verify` first. It runs the exact same readiness checks the
real configuration will, without changing anything. Catching a missing
SQL login or a missing Full-Text feature here costs you nothing; hitting
it mid-configuration costs you a partial, half-applied state to clean up.

```powershell
& $tfsConfig unattend /configure /unattendfile:'C:\Windows\Temp\ado-unattend.ini' /verify
```

Fix whatever it reports (see Troubleshooting below for the specific
errors this build hit), re-run `/verify` until it's clean, then run the
real thing:

```powershell
& $tfsConfig unattend /configure /unattendfile:'C:\Windows\Temp\ado-unattend.ini'
```

A clean run ends with `ServerConfiguration completed successfully.` and
exit code `0`. It configures IIS, creates the configuration database,
stands up both websites, installs the background services, and creates
your first project collection, all in one pass.

**Immediately after**: delete the ini file (it has the plaintext
password), and detach the install ISO.

## Step 8: Add the new databases to your Availability Group

**Do not skip this step even though the server works fine without it.**
This is the step this build actually skipped the first time, and it
caused a real outage hours later, the same day. See the Troubleshooting
section for the full story of what that looked like. Do it right after
Step 7 succeeds, not "eventually."

```sql
-- On whichever node is currently PRIMARY:
ALTER DATABASE [AzureDevOps_LabOpsConfiguration] SET RECOVERY FULL;
ALTER DATABASE [AzureDevOps_LabOpsDefaultCollection] SET RECOVERY FULL;

BACKUP DATABASE [AzureDevOps_LabOpsConfiguration]
    TO DISK = 'C:\SomeLocalOrSharedPath\AzureDevOps_LabOpsConfiguration_Full.bak'
    WITH INIT, COMPRESSION;
BACKUP DATABASE [AzureDevOps_LabOpsDefaultCollection]
    TO DISK = 'C:\SomeLocalOrSharedPath\AzureDevOps_LabOpsDefaultCollection_Full.bak'
    WITH INIT, COMPRESSION;

ALTER AVAILABILITY GROUP [YourAGName] ADD DATABASE [AzureDevOps_LabOpsConfiguration];
ALTER AVAILABILITY GROUP [YourAGName] ADD DATABASE [AzureDevOps_LabOpsDefaultCollection];
```

If your AG has Automatic Seeding enabled, the `ADD DATABASE` step alone
pushes the data to every secondary, no manual backup/restore per
replica needed. Check first:

```sql
SELECT ar.replica_server_name, ar.seeding_mode_desc FROM sys.availability_replicas ar;
```

`AUTOMATIC` means you're set; `MANUAL` means you'll need to back up and
restore onto each secondary yourself before the `ADD DATABASE` will
succeed.

**Verify it actually joined and actually synchronized.** Don't just
trust that the command didn't error:

```sql
SELECT ar.replica_server_name, d.database_name, drs.synchronization_state_desc, drs.is_suspended
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id
JOIN sys.availability_databases_cluster d ON drs.group_database_id = d.group_database_id
WHERE d.database_name LIKE 'AzureDevOps%';
```

Every row for every replica should say `SYNCHRONIZED` with
`is_suspended = False`. If a database is missing from this list
entirely, it isn't in the AG. Go back and redo the `ADD DATABASE` step.

## Step 9: Wire up sign-in with Active Directory groups

Azure DevOps Server authenticates directly against the domain, no
separate accounts to create. What you're actually configuring is which
AD identities get mapped into Azure DevOps's own built-in security
groups.

The tool is `TFSSecurity.exe`, in the same `Tools` folder as
`TfsConfig.exe`. **List the real group names before trying to reference
them.** The display names have a scope prefix that's easy to guess
wrong:

```powershell
$tfssec = 'C:\Program Files\Azure DevOps Server\Tools\TFSSecurity.exe'
& $tfssec /g /collection:https://yourhostname/DefaultCollection
```

That dumps every built-in group on the collection, several dozen of
them, most of which you don't need. Pipe it through a filter for the
handful that actually matter at this scope:

```powershell
& $tfssec /g /collection:https://yourhostname/DefaultCollection | Select-String 'Administrators'
```

Collection-level groups show up as `[DefaultCollection]\<name>` (not
`[TeamFoundation]\<name>`, that prefix doesn't exist here despite
looking plausible). Contributors and Readers don't exist at this
scope at all, they're project-level groups; to see them, and a
project's own Administrators, you need the project's
`vstfs:///Classification/TeamProject/<project-guid>` URI, not its
friendly URL path:

```powershell
& $tfssec /g "vstfs:///Classification/TeamProject/<project-guid>" /collection:https://yourhostname/DefaultCollection
```

Same filtering trick applies here, `Select-String 'Administrators|Contributors|Readers'` picks the ones you're actually nesting groups into out of the project's own full list.

Then nest your AD groups in. The `n:` prefix tells `TFSSecurity` the
value that follows is a Windows account or group name to resolve
against AD, not an Azure DevOps identity, `DOMAIN\` has to be your
actual NetBIOS domain name, not a placeholder to leave as-is:

```powershell
& $tfssec /g+ "[DefaultCollection]\Project Collection Administrators" "n:DOMAIN\ADO-Collection-Admins" /collection:https://yourhostname/DefaultCollection
& $tfssec /g+ "[YourProject]\Contributors" "n:DOMAIN\ADO-YourProject-Contributors" /collection:https://yourhostname/DefaultCollection
& $tfssec /g+ "[YourProject]\Readers" "n:DOMAIN\ADO-YourProject-Readers" /collection:https://yourhostname/DefaultCollection
```

**Verify a specific user's resolved access before trusting the group
nesting alone**:

```powershell
& $tfssec /imx "n:DOMAIN\someuser" /collection:https://yourhostname/DefaultCollection
```

The output lists every group that user is a direct or indirect member
of. Confirm the ones you expect are actually there.

## Step 10: Create a project and repository

No interactive desktop session needed, the REST API does this in two
calls, authenticated the same way the rest of this build has been:
Windows Integrated Authentication against whatever domain identity is
running the script, no separate credential prompt or token to manage.
Run this as (or under a scheduled task running as) an account with
Project Collection Administrator rights, and `-UseDefaultCredentials`
handles the rest. Creation is asynchronous, so poll the returned
operation until it reports success:

```powershell
$body = @{
    name = 'YourProject'
    capabilities = @{
        versioncontrol   = @{ sourceControlType = 'Git' }
        processTemplate  = @{ templateTypeId = 'adcc42ab-9882-485e-a3ed-7678f01f66bc' }  # Agile
    }
} | ConvertTo-Json -Depth 5

$r = Invoke-WebRequest -Uri 'https://yourhostname/DefaultCollection/_apis/projects?api-version=7.0' `
    -Method POST -Body $body -ContentType 'application/json' -UseDefaultCredentials -UseBasicParsing
$opId = ($r.Content | ConvertFrom-Json).id

do {
    Start-Sleep -Seconds 5
    $status = (Invoke-WebRequest -Uri "https://yourhostname/DefaultCollection/_apis/operations/$opId?api-version=7.0" -UseDefaultCredentials -UseBasicParsing).Content | ConvertFrom-Json
} while ($status.status -notin 'succeeded','failed')

Write-Host $status.status
```

A default Git repository matching the project name is created
automatically. Confirm it:

```powershell
Invoke-WebRequest -Uri "https://yourhostname/DefaultCollection/YourProject/_apis/git/repositories?api-version=7.0" -UseDefaultCredentials -UseBasicParsing
```

![LabOps project overview, signed in with a domain account]({{ '/assets/img/gallery/azure-devops-server-labops-overview.png' | relative_url }})
_Signed in from WKS01 with a normal domain account, no separate Azure DevOps password prompted_

![test.labuser's own sign-in to the same project]({{ '/assets/img/gallery/azure-devops-server-labops-testlabuser.png' | relative_url }})
_A second domain identity, same project, same story: its own sign-in, no separate password_

![The LabOps Git repo, created via the REST API]({{ '/assets/img/gallery/azure-devops-server-labops-repo.png' | relative_url }})
_The repository created above, with its clone URL ready to go_

## Day-2: checking system health

**Is the site actually up?**

```powershell
Import-Module WebAdministration
Get-Website | Select-Object Name, State
Get-ChildItem IIS:\AppPools | Select-Object Name, State
Get-Service -Name TFSJobAgent | Select-Object Name, Status
```

All should show `Started`/`Running`. `TFSJobAgent` handles background
work (permission sync, notifications, scheduled jobs). If it's stopped,
things like AD group membership changes won't propagate into Azure
DevOps until it's running again.

**Are the databases healthy and actually in the AG?** Re-run the query
from Step 8 periodically, or after any SQL-side maintenance (patching,
failovers, memory reconfiguration). Anything that touches the AG is
worth a quick recheck that both databases are still `SYNCHRONIZED`.

**Application-level errors**: the Windows Application event log on the
application-tier server, filtered to provider `TFS Services`, is where
Azure DevOps Server logs its own detailed exceptions: full stack
traces, inner exceptions, and the actual SQL error underneath a generic
web error message. Always check here first, before guessing.

```powershell
Get-WinEvent -LogName 'Application' -MaxEvents 50 |
    Where-Object ProviderName -eq 'TFS Services' |
    Select-Object TimeCreated, LevelDisplayName, Message
```

## Troubleshooting

### "The setting 'ADDLOCAL' specified is not recognized" when adding Full-Text Search

You're using the wrong parameter for this SQL Server Setup version.
Use `/FEATURES=` and list the engine alongside the new feature; Setup
won't treat a bare feature name as an add-on to an existing instance
without it:

```powershell
D:\setup.exe /ACTION=Install /FEATURES=SQLEngine,FullText /INSTANCENAME=MSSQLSERVER /IACCEPTSQLSERVERLICENSETERMS /Q
```

**If this is an Availability Group**, do this one node at a time:
secondaries first, checking AG health is fully healthy after each
before moving to the next, then a planned manual failover to move the
primary role off before patching the last node:

```sql
-- Check before/after each node:
SELECT replica_server_name, role_desc, synchronization_health_desc
FROM sys.dm_hadr_availability_replica_states ars
JOIN sys.availability_replicas ar ON ars.replica_id = ar.replica_id;

-- Planned failover to reach the current primary safely:
ALTER AVAILABILITY GROUP [YourAGName] FAILOVER;   -- run on the target secondary
```

### `Unable to bind the certificate for the following website binding` during `/verify` or `/configure`

Something else on this host already owns a non-SNI certificate binding
on the same port. Check:

```powershell
netsh http show sslcert
Get-WebBinding | Select-Object bindingInformation, protocol, sslFlags
```

If you see an existing `*:443:` binding (empty host header, `sslFlags:
0`) for a different site, that's the conflict. HTTP.sys can only bind
one certificate per IP:port without SNI. Fix by enabling SNI on the
existing binding and re-registering it explicitly with the
hostname-based syntax (the IIS-level SNI toggle alone doesn't always
propagate down to HTTP.sys automatically):

```powershell
Set-WebBinding -Name 'ExistingSiteName' -BindingInformation '*:443:existing.hostname' -PropertyName sslFlags -Value 1
netsh http delete sslcert ipport=0.0.0.0:443
netsh http add sslcert hostnameport=existing.hostname:443 certhash=<thumbprint> certstorename=MY appid='{<any-guid>}'
```

Confirm the existing site still works before re-running Azure DevOps
Server's configuration.

### `TF246017: Azure DevOps Server could not connect to the database`, with an inner `Login failed for user` error

![The actual outage: a 500 error at the collection root]({{ '/assets/img/gallery/azure-devops-server-tf246017-outage-collection.png' | relative_url }})
_What this actually looked like from a browser, from skipping Step 8_

![The same outage, 25 seconds later, at the LabOps project URL]({{ '/assets/img/gallery/azure-devops-server-tf246017-outage-labops.png' | relative_url }})
_Every page hit it, not just one, which is what made this look worse than a single bad request_

Read past the headline error before assuming it's a permissions problem.
The full exception in the Application event log will show the actual
SQL error underneath: this build's said `Cannot open database
"AzureDevOps_LabOpsConfiguration" requested by the login`, SQL error
4060/18456. That's not necessarily "this account can't log in." It can
also mean **the database itself isn't reachable from wherever the
connection is currently being routed**, which is exactly what happened
here.

**The actual cause, and the one worth checking first if you're on an
AG**: the database was never added to the Availability Group (Step 8
above), so it only exists on whichever node happened to be primary when
it was created. If the AG's primary role has since moved, through a
failover, planned or not, the listener now points at a node that
doesn't have the database at all.

Confirm this is what's happening:

```sql
-- Run against each node individually, not through the AG listener:
SELECT name, state_desc FROM sys.databases WHERE name LIKE 'YourApp%';
```

If the database shows up on one specific node and nowhere else, that's
your answer. Fail the AG back to that node (or move the databases
properly, see Step 8) rather than troubleshooting the service account,
which is very likely fine.

### A page loads other Azure DevOps pages fine but one specific project page times out

Check the Application event log for genuine errors at that exact
timestamp first. An empty log at that timestamp, combined with the site
otherwise responding normally elsewhere, points at a transient issue
with however you're testing (a proxy, a scripted request without proper
session handling) rather than the server itself. Confirm from an actual
browser, on an actual client machine, before spending time chasing a
server-side cause that may not exist.

### A user isn't seeing the access you expect

Don't just check that their AD group is nested in the right Azure DevOps
group. Verify their *resolved* effective membership directly:

```powershell
& $tfssec /imx "n:DOMAIN\theuser" /collection:https://yourhostname/DefaultCollection
```

Common reasons this doesn't match expectations: the AD group nesting is
one level too shallow (nested a group into a group that isn't actually
the one granting access), or `TFSJobAgent` is stopped and hasn't synced
the AD group membership change yet. Check its status per the Day-2
section above.

## Division of labor

The agent: the entire install from media selection through
`tfsconfig unattend`, all the troubleshooting (SNI conflict, the SQL
Server feature install across three nodes, the orphaned-database
diagnosis and fix), the AD group and Azure DevOps permission wiring, and
the project/repository creation. Me: catching the actual outage with a
real screenshot when the automated checks all looked fine, approving
each SQL-AG-touching step before it ran, and the call to keep the
application-tier host on its current OS version and source matching
media rather than downgrade it.

## What's next

The platform itself is live and working. Still ahead: a build agent on
the same host, actual pipeline definitions once there's real content to
build, and eventually migrating other repositories into this server now
that it's proven out. Before any of that, though, next up is standing
up SCOM 2025 across the whole environment, covered in the next post.

![Work items / Boards, live and empty]({{ '/assets/img/gallery/azure-devops-server-work-items-empty.png' | relative_url }})
_Boards are reachable and ready; nothing tracked here yet_

![Pipelines waiting for a build agent]({{ '/assets/img/gallery/azure-devops-server-pipelines-empty.png' | relative_url }})
_Same story for Pipelines: the platform side is done, the actual automation is next_
