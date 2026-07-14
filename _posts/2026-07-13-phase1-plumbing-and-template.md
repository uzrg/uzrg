---
title: Homelab Build-Out — Phase 1, OS Install the Hard Way, Then Building a Repeatable OS Template
author: uzrg
date: 2026-07-13 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Microsoft, HyperV, Windows Server]
tags: [Microsoft, HyperV, Windows, Windows Server, Automation]
pin: false
---

# Phase 1: the hard way, first

Phase 1 covers the two boxes the rest of SUPERLAB depends on: FS01 and
DHCP01. The Phase 0 survey had already warned this would be a bigger lift
than planned — both were blank installs, not the configured servers the
inventory claimed. What follows is an honest account of how that went,
failures included, and a clear line throughout between what the agent
handled on its own and where I had to step in myself.

## DHCP01: a pre-staged account is not a shortcut

DHCP01 needed a static IP before it could even resolve the domain — nothing
had ever assigned it one, and a DHCP server can't obtain its own address
from itself. Once that was in place, the domain join kept failing with "the
account already exists": a computer object for DHCP01 had been pre-staged
in AD back in April and never touched since.

The agent tried resetting its password first. That did nothing — Active
Directory's join logic doesn't quietly adopt an existing enabled account
just because the password now matches. The object itself had to go before a
real join could create a proper one in its place, which meant asking me to
authorize the deletion: removing AD computer objects is one of the actions
I've deliberately fenced off behind explicit approval, not something the
agent decides on its own. I approved it — twice, since the identical problem
resurfaced on FS01's own pre-staged object. Lesson: a pre-staged account
only helps if whatever provisions the machine actually knows to reuse it.
Otherwise it's a landmine waiting for the first real join.

## FS01: three ways an unattended install goes quiet

FS01 had no operating system at all, just an empty disk and a mounted
installer ISO. Automating that install surfaced three distinct ways Windows
Setup can stall without a sound — and on FS01, each one needed me at the
console before the agent could continue.

**The boot loader waits for a keystroke.** Windows install media shows a
"Press any key to boot from DVD..." prompt before Setup even starts,
deliberately, so a machine doesn't reinstall itself in a loop with the disc
left in the drive. On a headless boot, nothing ever presses that key, and
the VM just sits there looking like a generic "no operating system"
failure. The agent tried one blind keystroke over Hyper-V's virtual
keyboard, then correctly declined to guess again — sending input at a
screen it can't see isn't something it should do speculatively. I opened
the console myself, watched for the prompt, and pressed the key.

**Setup asks for a product key if it doesn't have one.** Without one in the
answer file, Setup drops out of the unattended path to ask interactively —
another silent stall, indistinguishable from the first without eyes on the
screen. I clicked through it by hand.

**A password baked into an answer file is a liability, not an
inconvenience.** The obvious way to fully automate this is embedding the
local Administrator password in the answer file, burned onto a small ISO
Setup reads automatically. It works — but the password then sits in
plaintext on a persistent artifact, and Windows later copies the entire
file to `C:\Windows\Panther\unattend.xml` on the installed system, also in
plaintext. The agent's own guardrails refused to write it, repeatedly,
regardless of how I told it to scope or word the justification — including
once when I asked it to grant itself an exception, which it also declined,
correctly treating that as a worse problem than the original one. We
settled on the simpler fix instead: leave the password out entirely. Setup
finishes unattended, the built-in Administrator ends up disabled, and I set
the real password once at the console before the machine can be joined.

## Turning three fixes into one template

Hitting all three on a single VM is reason enough not to repeat the
exercise on every future box. So the agent built a proper, reusable,
generalized Windows Server 2025 Standard template, engineered to clear each
problem without me:

- **Product key:** Microsoft publishes generic AVMA keys for exactly this
  scenario. Baking the right one into the answer file means Setup never
  stops to ask.
- **The boot-loader prompt:** instead of waiting on me, the agent injects
  the keystroke itself — Hyper-V exposes a virtual keyboard over WMI, so a
  script sends a burst of keypresses in the first few seconds after boot,
  reliably catching the narrow window the prompt stays open.
- **The password:** the same fix as before, still mine to perform — skip it
  in the answer file, set it once at the console.

Two of the three stalls became fully automated; the third stayed a
deliberate manual step by design, not an oversight. With those in place,
the template's install ran start to finish without me until that one
moment.

## The generalization step most guides skip

Once Windows is installed the way you want it, `sysprep /generalize` strips
the machine-specific identity so the image is safe to clone: a new SID,
reset activation state, and so on. What it doesn't do — and what most
templating guides never mention — is reset the Windows **MachineGuid**
(`HKLM\SOFTWARE\Microsoft\Cryptography\MachineGuid`). That value survives
generalization and would otherwise be identical on every clone, a real
problem for anything using it as a uniqueness anchor — MECM's client
identity among them, which matters given ConfigMgr is further down this
lab's roadmap.

The agent's fix: a command embedded in the sysprep answer file's specialize
pass, running with SYSTEM rights a plain admin session doesn't have,
deleting the registry value outright. Windows regenerates a fresh one the
next time it's needed. Because the command lives inside the generalized
image itself, it reruns on every future clone's first boot too — each one
gets its own MachineGuid automatically, no extra step, no me required.

## Where this leaves things

FS01 and DHCP01 are both up, domain-joined, and doing their Phase 1 jobs.
Alongside them sits a sysprepped, generalized Windows Server 2025 Standard
template — AVMA-keyed, boot-prompt-proofed, MachineGuid-clean — ready for
whatever gets built next, and needing exactly one thing from me: a
password, once, at the console.
