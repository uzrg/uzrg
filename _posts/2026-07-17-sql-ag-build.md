---
title: Homelab Build-Out — Three SQL Nodes Overnight, and What an AI Agent's Guardrails Look Like in Practice
author: uzrg
date: 2026-07-17 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Microsoft, HyperV, Windows Server]
tags: [Microsoft, Windows Server, SQL Server, AlwaysOn, WSFC, gMSA, Automation, AI Agent]
pin: false
---

# The data tier, built overnight

The lab's data tier is live: three Windows Server 2025 nodes in a
failover cluster (SQLCLU01) running a SQL Server 2022 Always On
availability group (SQLAG01), with a listener that passed a planned
failover test — primary lost and restored, same name answering
throughout, two-minute round trip. Configuration Manager, SCOM, RDS,
and WSUS will all sit on this tier.

Most of the work was done overnight by an autonomous operations agent
running inside explicit guardrails, on one instruction given before I
went to bed: start the SQL Server build. This post is the build log,
and a field report on how the guardrails held — including the points
where they said no.

## Starting point

Three identical VMs with clean Windows Server 2025 Standard installs:
workgroup, no IP configuration, four raw data disks each, and two NICs
each — one on the servers VLAN, one on a VLAN provisioned for cluster
traffic. The SQL Server 2022 Enterprise media and key were staged, and
the file server already exposed the backup and witness shares from
Phase 1. Nothing to untangle.

## A better domain-join method

Phase 1 concluded that the lab's pre-staged computer accounts (created
in April, never used) block domain joins and must be deleted. The
overnight build proved that conclusion wrong. The agent's deletion
attempt was refused — AD object deletion requires my explicit sign-off,
and I was asleep — so it found another way: rename the machine to its
target name, reboot, then run a standard domain join with an account
that has full rights on the pre-staged object. The join adopts the
existing account instead of colliding with it.

That method is better than deletion, not just a workaround. Adopting
the account preserves its SID, and with it every group membership
granted in advance. The gMSA security groups populated in Phase 2 —
against computer objects for servers that did not exist yet — paid off
exactly as intended: all three SQL nodes retrieved their managed
service accounts on their first domain boot. The agent verified it
(`Test-ADServiceAccount` returned true on each node) before counting it
done.

## What the permission layer refused, and why that is the point

Three operations were blocked overnight:

1. **Deleting the pre-staged AD objects** — which forced the better
   join method above.
2. **Formatting the data disks** — destructive operations wait for a
   human, even when the target disks are verifiably raw and empty.
3. **Creating the failover cluster** — WSFC changes are on my
   confirmation-required list, with no exception for a legitimate
   build plan.

The agent's response to each refusal matters more than the refusal
itself: it never retried variations to slip past a block. It either
found a genuinely less-privileged path (the join), or staged the
blocked step as a reviewed, ready-to-run script and moved on. By
morning, the blocked work was a short, ordered checklist with one
script per step — and everything unblocked was already done and
verified: networking on both VLANs, activation, domain joins,
clustering features, share permissions for the backup path, media
attached, cluster validation passed.

An agent that can be handed an overnight build and trusted to stop at
exactly the lines you drew is worth more than either capability alone.

## The build

The morning session ran the four staged scripts in order. Two failures
along the way, both worth recording:

- **Volumes.** Each node: E: data, F: log, G: tempdb, H: local backup —
  NTFS, 64K allocation units. The first run failed on a detail nobody
  plans for: the attached SQL ISO had auto-claimed E: inside each
  guest, the exact letter intended for the data volume. The script was
  revised to be idempotent and to park the DVD at Z: before touching
  disks; the rerun was clean, twelve volumes verified.
- **Cluster.** Validation had already passed overnight (zero failures,
  two advisories). SQLCLU01 came up with all three nodes, the dedicated
  VLAN correctly auto-classified as cluster-only traffic, and a
  file-share witness on the Phase 1 file server for quorum.
- **SQL Server 2022 Enterprise.** Unattended installs, default
  instances. Because the groundwork predates the servers, the engine
  and agent services run under group managed service accounts from
  their very first start — no passwords anywhere in the configuration,
  no migration step later.
- **Availability group.** SQLAG01 spans all three nodes over the
  dedicated replication VLAN: two synchronous replicas with automatic
  failover, a third synchronous replica on manual failover, automatic
  seeding (the seed database synchronized on all replicas within
  seconds). Two failures here: a T-SQL syntax error in the agent's own
  script (a missing WITH in each replica clause — its scripts get code
  review like anyone else's), and the classic listener failure, because
  the cluster's computer account had no rights to create the listener's
  AD object. The broad fix — create rights on the entire Computers
  container — was refused by the permission layer. The narrow fix went
  through: pre-stage the listener object disabled and grant the cluster
  account control over that one object. The restrictive path is also
  simply the better practice.

Final verified state: listener answering on its own DNS name and
routing to the primary, all replicas healthy and synchronized, and the
seed database's initial backup on the file server share, written by the
gMSA.

## The failover test, with receipts

The last step ran with me watching: a planned failover to the second
node and back. One deliberate method note — no VM checkpoints for this
test. Restoring a single node's checkpoint under a live availability
group would desynchronize the group; the correct rollback path for a
planned failover is the failback itself.

The SQL error logs tell the story better than any dashboard screenshot.
The outgoing primary, at the moment of failover:

```
01:45:08  The state of the local availability replica in availability
          group 'SQLAG01' has changed from 'PRIMARY_NORMAL' to
          'RESOLVING_NORMAL'. ... the availability group is failing over
          to another SQL Server instance.
01:45:15  ... changed from 'RESOLVING_NORMAL' to 'SECONDARY_NORMAL'.
```

And the incoming primary, in the same second:

```
01:45:15  Always On: The local replica of availability group 'SQLAG01'
          is preparing to transition to the primary role.
01:45:15  ... changed from 'PRIMARY_PENDING' to 'PRIMARY_NORMAL'.
01:45:15  A connection ... to 'SQL01' has been successfully established.
01:45:15  A connection ... to 'SQL03' has been successfully established.
```

After the swap, the listener answered from the new primary and every
replica reported healthy. The failback ran the same way in reverse —
the original primary back in charge two minutes after the test began:

```
01:47:00  ... changed from 'PRIMARY_PENDING' to 'PRIMARY_NORMAL'.
```

Every database stayed synchronized on all three replicas at every
checkpoint of the test. That two-minute round trip is the deliverable:
the data tier behind Configuration Manager, SCOM, RDS, and WSUS can
lose its primary and keep answering on the same name.

## Division of labor

The agent: the survey, all three node preparations and domain joins,
the feature installs, share permissions, media staging, validation, the
cluster, the SQL installs, the availability group, every verification
step, and the scripts and runbook behind this post. Me: the morning
go-ahead and the approvals the guardrails required — disk formatting,
cluster creation, AD permission changes. Nothing on the domain joins,
because the agent found a path that never needed the dangerous
operation.

Three blank VMs became a healthy availability group in one unsupervised
night and one supervised morning — and four later phases now have their
data tier.
