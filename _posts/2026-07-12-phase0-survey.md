---
title: SUPERLAB Build-Out - Phase 0 Survey Results
author: uzrg
date: 2026-07-12 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Microsoft, HyperV, Windows Server]
tags: [Microsoft, HyperV, Windows, Windows Server, Active Directory]
pin: false
---

# Phase 0: surveying before building

Before touching anything in the next phase of the SUPERLAB build-out, I
ran a read-only survey of every VM in the lab — what's actually running,
what's actually configured, and whether Active Directory is healthy enough
to build on top of. No changes were made this pass; the goal was just an
honest inventory.

## Active Directory: healthy

The two domain controllers passed every standard health check: `dcdiag`,
replication (`repadmin /showrepl` and `/replsummary`), SYSVOL/DFSR
replication state, FSMO role consistency, and time sync. Zero replication
failures, low latency between the two. This was the go/no-go gate for
everything else, and it's a clean pass.

One small thing turned up: one of the configured DNS forwarders isn't
answering queries. Doesn't affect internal domain name resolution at all,
but it's on the list to sort out before the next phase.

## Everything else: earlier than expected

Here's the useful part of doing a real survey instead of assuming the
plan matches reality: most of the VMs that were expected to already carry
a role turned out to be further back than that. Some are booted, generic
Windows installs that haven't been joined to the domain or had a role
installed yet. A few others haven't had an operating system installed at
all yet — the install media is sitting there mounted, waiting.

None of this is a problem — it's exactly what a Phase 0 survey is for. It
just means the next phase's actual to-do list is "build this from
scratch" in more places than the original plan assumed, rather than
"reconcile the existing config." Better to find that out now, read-only,
than halfway through a phase.

## What's next

Write-up went to the operator for review. Once the build order is
confirmed, Phase 1 starts with the core plumbing (file services + DHCP),
working outward from the two healthy domain controllers.
