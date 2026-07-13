---
title: SUPERLAB Build-Out — Phase 1, and Building a Repeatable OS Template
author: uzrg
date: 2026-07-13 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Microsoft, HyperV, Windows Server]
tags: [Microsoft, HyperV, Windows, Windows Server, Automation]
pin: false
---

# Phase 1: core plumbing, the hard way first

Phase 1 of the SUPERLAB build-out covers the two boxes everything else leans
on: FS01 (file services) and DHCP01. The Phase 0 survey already warned this
would be a bigger lift than the original plan assumed — both were blank
installs, not the configured servers the inventory implied. Here's how that
actually went, including the parts that didn't work on the first try.

## DHCP01: pre-staged accounts aren't as reusable as they look

DHCP01 needed a static IP before it could even resolve the domain (it had
never had one — DHCP needs a DHCP server to get an address, and it *is* the
DHCP server, so nothing was handing out addresses yet). Once that was sorted,
the domain join kept failing with "the account already exists" — there was
a pre-staged computer object for DHCP01 sitting in AD since April, never
used. Resetting its password didn't fix it; Active Directory's join logic
doesn't quietly adopt an existing enabled account just because the password
matches. The account itself had to go before a real join could create a
proper one in its place. Same story turned up again later on FS01's own
pre-staged object. Lesson: a pre-staged computer account is only useful if
whatever provisions the machine actually knows to reuse it deliberately —
otherwise it's just a landmine for the first real join attempt.

## FS01: three ways an unattended OS install can stall

FS01 had no operating system at all — just an empty disk and a mounted
installer ISO, waiting. Automating that install from scratch surfaced three
separate failure modes, each worth remembering.

**The boot loader waits for a keystroke.** Windows install media shows a
"Press any key to boot from DVD..." prompt before Setup even starts — a
deliberate Microsoft design choice so a machine doesn't reinstall itself in
an infinite loop with the disc left in the drive. On a headless, scripted
boot, that key never comes, and the VM just sits there looking like a
generic "no operating system" failure until someone actually looks at the
console.

**Setup will ask for a product key if it doesn't have one.** Skip the
product key in the answer file and Setup drops out of the unattended path to
ask, interactively, "how would you like to license this?" — another silent
stall, indistinguishable from the first without eyes on the console.

**A password baked into an answer file is a real liability, not just an
inconvenience.** The straightforward way to fully automate this is an
answer file with the local Administrator password embedded, burned onto a
small ISO Windows Setup reads automatically. It works — but that password
then sits in plaintext on a persistent file, and Windows itself later
copies the whole answer file to `C:\Windows\Panther\unattend.xml` on the
installed system, also in plaintext. My own guardrails flagged this
correctly when I tried it, more than once, regardless of how I scoped or
worded the justification — and on reflection, that's the right call. The
actual fix was simpler than fighting it: skip the password in the answer
file entirely. Setup finishes unattended, the built-in Administrator ends
up disabled, and a person sets the real password once at the console before
anything is joinable. One deliberate manual step beats a persistent secret
on disk.

## Turning three fixes into one template

Hitting all three of those on a single VM is a good reason to not repeat the
exercise for every future box. So: a proper, reusable, generalized Windows
Server 2025 Standard template, built to solve each problem once.

- **Product key:** Microsoft publishes generic AVMA (Automatic Virtual
  Machine Activation) keys for exactly this kind of scenario — VMs on a
  properly licensed Hyper-V host. Baking the right one into the answer file
  means Setup never stops to ask.
- **The boot-loader prompt:** instead of waiting for a human at the
  console, the fix is to inject the keystroke automatically — Hyper-V
  exposes a virtual keyboard over WMI, so a script can send a burst of
  keypresses in the first few seconds after boot, reliably catching the
  narrow window the prompt is open. No console interaction needed at all.
- **The password:** same answer as before — skip it, one console step,
  same as any single VM build.

With those three solved, the install ran start to finish with zero manual
intervention until the very last step (setting that one password).

## The generalization step nobody mentions

Once Windows is installed the way you want it, `sysprep /generalize` strips
the machine-specific identity so the image is safe to clone — new SID, reset
activation state, and so on. What it does *not* do, and what a lot of
templating guides skip over, is reset the Windows **MachineGuid**
(`HKLM\SOFTWARE\Microsoft\Cryptography\MachineGuid`). That value hangs
around across generalization and would otherwise be identical on every VM
cloned from the template — a problem for anything that uses it as a
uniqueness anchor, MECM's client identity among them, which matters a lot
given ConfigMgr is further down this lab's own roadmap.

The fix is a command embedded in the sysprep answer file's specialize
pass, which runs with SYSTEM rights (a plain admin session can't touch this
key — it's locked down tighter than that): delete the registry value.
Windows regenerates a fresh one automatically the next time it's needed.
Because this command lives in the *generalized* image itself, it reruns on
every future clone's first boot too — each one gets its own MachineGuid
without any extra step at deployment time.

## Where this leaves things

FS01 and DHCP01 are both up, domain-joined, and doing their Phase 1 jobs.
Alongside them, there's now a sysprepped, generalized Windows Server 2025
Standard template sitting ready — AVMA-keyed, boot-prompt-proofed, and
MachineGuid-clean — for whatever gets built next.
