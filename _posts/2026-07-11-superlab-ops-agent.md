---
title: Bringing an Operations Agent Into SUPERLAB
author: uzrg
date: 2026-07-11 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Microsoft, HyperV, Windows Server]
tags: [Microsoft, HyperV, Windows, Windows Server, Automation]
pin: false
---

# Why an agent, and why now

SUPERLAB has grown past the point where I can hold the whole build-out in
my head between sessions — two domain controllers, a SQL Always On tier,
an RDS farm, MECM, SCOM, DHCP, all wired together across VLANs behind
pfSense. Before starting the next round of build work, I set up
Claude Code as a standing operations agent that runs directly on the
Hyper-V host, so every session starts from the same shared understanding
of what the lab is, what's safe to touch, and what isn't.

## The operating charter

Everything starts from a single `CLAUDE.md` at the root of the ops repo.
It's not a prompt so much as a charter: what SUPERLAB is, the full VM
inventory and each box's intended role, the phase-by-phase build-out
roadmap, and — most importantly — the guardrails. Some highlights:

- Read-only by default. Anything touching the domain controllers, the SQL
  AG, or MECM needs explicit confirmation before it happens.
- pfSense is read-only, permanently. The agent can SSH in to read config
  for context, but it never changes anything there — any firewall or DHCP
  change it identifies gets flagged back to me instead.
- Checkpoint before any risky guest change, every time, no exceptions.
- Never take down more than one Always On AG node, or both domain
  controllers, at once.

Live state always wins over what's written down here — if the lab has
drifted from the plan, the agent is expected to say so rather than assume
the document is still accurate.

## A shared vocabulary, not raw cmdlets

Rather than having the agent improvise Hyper-V and Active Directory
commands from scratch every session, I built a small PowerShell module —
`LabOps` — that wraps the handful of operations it actually needs into
consistently-named functions: a one-screen state table across every VM,
a checkpoint-first snapshot helper with a standard naming convention, a
connectivity matrix, and a wrapper around PowerShell Direct for reaching
into guests.

PowerShell Direct turned out to be the right primary channel for guest
access. It rides the VMBus rather than the network, so it keeps working
even when a guest's networking, WinRM, or firewall is broken — which
matters a lot when half the job is figuring out why a VM is unreachable
in the first place.

## Playbooks as skills

The repeatable procedures live as Claude Code skills rather than as
one-off instructions I retype each time: a full Phase 0 inventory survey
with an AD health gate as its go/no-go check, a lab-wide health check that
writes a dated report, and a checkpoint-first wrapper for any change that
touches VM state. Each one encodes the specific verification steps for
that kind of work — the health check knows to check AG synchronization
state and MECM component status, the VM lifecycle skill knows to fail over
before touching an AG primary — so the judgment calls are made once, in
writing, instead of re-derived under pressure every session.

## Guardrails enforced at the tool level

Documentation is one layer; tool permissions are another. The read-only
diagnostic commands the agent uses constantly — `dcdiag`, `repadmin`,
`w32tm`, the various `Get-*`/`Test-*` cmdlets — are pre-approved so the
agent isn't stopped for permission on every single inspection step.
Destructive operations — removing a VM, deleting an AD object, dropping a
DHCP scope, formatting a volume — are explicitly denied at the settings
level, not just discouraged in prose. Belt and suspenders.

## First real test: Phase 0

The first thing I had the agent do with this setup was exactly what it's
for: a full, read-only Phase 0 survey of the entire lab before any build
work starts. That write-up is next.
