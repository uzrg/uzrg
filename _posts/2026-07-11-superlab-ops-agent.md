---
title: Bringing an AI Agent Into the Homelab
author: uzrg
date: 2026-07-11 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Microsoft, HyperV, Windows Server]
tags: [Microsoft, HyperV, Windows, Windows Server, Automation, AI]
pin: false
---

# Why an AI agent, and why now

SUPERLAB has grown past the point where I can hold the whole build-out in
my head between sessions — two domain controllers, a SQL Always On tier,
an RDS farm, MECM, SCOM, DHCP, all wired together across VLANs behind
pfSense. Before starting the next round of build work, I set up
Claude Code — an AI coding agent, not a script or a fixed automation
pipeline — to run directly on the Hyper-V host and do the actual legwork:
running diagnostics, surveying VMs, drafting reports, even drafting the
blog posts on this site. I'm upfront about that because it matters for
how you should read anything published here going forward: when a post
describes a survey being run or a finding being made, that work was done
by the agent, not by me typing commands by hand. My role is to write the
rules it operates under, decide what it's allowed to touch, and review
and approve what it produces before anything ships — including this post,
which the agent drafted and I edited.

## The operating charter

Everything the agent does starts from a single `CLAUDE.md` file at the
root of the ops repo, which I wrote and maintain. It's not a prompt so
much as a charter: what SUPERLAB is, the full VM inventory and each box's
intended role, the phase-by-phase build-out roadmap, and — most
importantly — the guardrails the agent operates inside. Some highlights:

- Read-only by default. Anything touching the domain controllers, the SQL
  AG, or MECM needs my explicit confirmation before it happens.
- pfSense is read-only, permanently. The agent can SSH in to read config
  for context, but it never changes anything there — any firewall or DHCP
  change it identifies gets flagged back to me instead of acted on.
- Checkpoint before any risky guest change, every time, no exceptions.
- Never take down more than one Always On AG node, or both domain
  controllers, at once.

Live state always wins over what's written down here — if the lab has
drifted from the plan, the agent is expected to say so rather than assume
the document is still accurate. That's a rule for the agent's own
reporting, too: it's expected to flag what it verified against a real
command versus what it's inferring, rather than blur the two.

## A shared vocabulary, not raw cmdlets

Rather than having the agent improvise Hyper-V and Active Directory
commands from scratch every session, the lab has a small PowerShell
module — `LabOps` — that wraps the handful of operations the agent
actually needs into consistently-named functions: a one-screen state
table across every VM, a checkpoint-first snapshot helper with a standard
naming convention, a connectivity matrix, and a wrapper around
PowerShell Direct for reaching into guests.

PowerShell Direct turned out to be the right primary channel for guest
access. It rides the VMBus rather than the network, so it keeps working
even when a guest's networking, WinRM, or firewall is broken — which
matters a lot when half the job is figuring out why a VM is unreachable
in the first place.

## Playbooks as skills

The repeatable procedures live as Claude Code skills rather than as
one-off instructions retyped every session: a full Phase 0 inventory
survey with an AD health gate as its go/no-go check, a lab-wide health
check that writes a dated report, and a checkpoint-first wrapper for any
change that touches VM state. Each one encodes the specific verification
steps for that kind of work — the health check knows to check AG
synchronization state and MECM component status, the VM lifecycle skill
knows to fail over before touching an AG primary — so the judgment calls
get made once, deliberately, by me, rather than re-derived by the agent
under pressure every session.

## Guardrails enforced at the tool level

Documentation is one layer; tool permissions are another. The read-only
diagnostic commands the agent uses constantly — `dcdiag`, `repadmin`,
`w32tm`, the various `Get-*`/`Test-*` cmdlets — are pre-approved so it
isn't stopped asking for permission on every single inspection step.
Destructive operations — removing a VM, deleting an AD object, dropping a
DHCP scope, formatting a volume — are explicitly denied at the settings
level, not just discouraged in prose. Belt and suspenders: I don't have to
trust the agent's judgment alone to keep it away from anything
irreversible.

## First real test: Phase 0

The first thing I had the agent do with this setup was exactly what it's
for: a full, read-only Phase 0 survey of the entire lab before any build
work starts — the agent ran every check, and I reviewed the findings
before signing off. That write-up is next.
