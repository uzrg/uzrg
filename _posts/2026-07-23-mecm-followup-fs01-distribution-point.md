---
title: Homelab Build-Out — MECM Follow-Up, An Unresolved Distribution Point Defect on FS01
author: uzrg
date: 2026-07-23 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Microsoft, HyperV, Windows Server]
tags: [Microsoft, ConfigMgr, MECM, SCCM, Troubleshooting, Distribution Point, AI Agent]
pin: false
mermaid: false
---

# Status: open, and worth publishing that way

The [last MECM post]({% post_url 2026-07-21-mecm-build-ha-boundaries-first-app %})
closed with the Yubico Smart Card Minidriver application "proven,
mostly." Two days later I had enough time to actually sit down and go
through the MECM console, and I noticed it showing a failed-to-distribute-content
error pointing to FS01. I captured the screenshot and pointed the agent
to it. What follows is the agent's investigation of that error: five
fix attempts, five identical failures, at which point I decided to
tell the agent to stop and publish rather than push for a sixth guess.
I'm writing it up unresolved on purpose: a team that only publishes
clean endings teaches the wrong lessons.

**Bottom line, for anyone skimming:** the defect is real, and it isn't
something introduced by recent work. It involves exactly one thing:
pushing new or updated content to clients through FS01's distribution
point. Root cause is still unidentified after five reasonable,
correctly executed fix attempts by the agent. Decision on remediation
path is pending, but I'm heavily leaning toward adjusting the
architecture by relocating the content library and distribution point
role to MECM01. Another experiment I plan to run: hand the agent a
screenshot from WKS01's Software Center and see whether it starts its
investigation by looking into the CCM logs. Before we get to that,
let's first review what the agent uncovered and the fixes it attempted.

## What the screenshots actually showed

The MECM console's Content Status view for the Yubico app read `Failed
to distribute content`, one asset in Error, FS01 named as the target.

<img src="{{ '/assets/img/gallery/mecm-yubico-content-distribution-error.png' | relative_url }}" alt="Yubico Smart Card Minidriver stuck in a content-distribution error against FS01">
_Content Status view: one asset, Error, FS01 as the target — the screenshot that started this investigation._

On the client, the same failure looks different. On WKS01, Software
Center showed the Yubico Smart Card Minidriver stuck at "Installing…,"
downloading, 0% complete — not an error message, just a spinner that
was never going to move.

<img src="{{ '/assets/img/gallery/mecm-yubico-deployment-no-content.png' | relative_url }}" alt="WKS01 Software Center showing the Yubico Smart Card Minidriver stuck installing, downloading at 0 percent">
_WKS01's Software Center: policy delivered, install triggered, content stuck at 0% — the same defect, seen from the client side._

## The agent's findings and fixes

On FS01's content library (`F:\SCCMContentLib`), every package ever
distributed there — the client package, the client upgrade package,
and the Yubico app — is stuck as an unfinalized `.temp` manifest in
`DataLib`, unchanged since the original build. The bytes transfer
fine; only the commit step that finalizes them never completes.

That reframes the previous post's "proven, mostly" result: detection
worked because the first push happened to land, not because the
distribution point was healthy. It's been silently failing every retry
since — exactly the kind of gap no dashboard catches on its own.

The underlying error, traced into FS01's own `smsdpprov.log`, was
consistent across every attempt:

```
CFileLibrary::AddFile failed; 0x80040154
CContentDefinition::AddFile failed; 0x80040154
Failed to add file 'YubiKey-Minidriver-5.0.4.273-x64.msi' to content library. Error code: 0X80040154
```

`0x80040154` is `REGDB_E_CLASSNOTREG` — a COM class that can't be
instantiated. A specific enough signature to act on. Not specific
enough, as it turned out, to fix on the first several tries:

1. **BITS was stopped on FS01.** Selected first because BITS is the
   transport ConfigMgr uses to push content to a DP over HTTP, and
   auditing FS01's DP prerequisites turned up `BITS-IIS-Ext` installed
   but the service itself never running (`StartMode: Manual`) — a
   real, independently worth-fixing gap regardless of whether it was
   the cause. Corrected and set to Automatic. No change to the
   underlying error.
2. **WIMGAPI, a standard DP prerequisite, was sitting as an uninstalled
   `.msi`** beside the real DP binaries. Selected because WIMGAPI
   registers COM components the content library's file operations
   depend on, and finding the installer still sitting unexecuted right
   next to the DP's real binaries made it the obvious next candidate.
   Installed cleanly with a clean exit code. No change.
3. **`smsdp.dll` re-registered via `regsvr32`.** Selected because the
   error is explicitly a COM registration failure, and `smsdp.dll` is
   the binary that implements the DP's local WMI provider — the most
   direct match to the error signature of anything on the box.
   Succeeded. No change.
4. **A full reboot of FS01.** Selected on the theory that a COM/WMI
   registration change (from either of the prior two fixes) might not
   take effect in an already-running process, and a fresh process
   space would pick it up. I required a health check before and after:
   FS01 is the file-share witness for `SQLCLU01`, not a voting data
   node, so the risk was real but bounded, and I wasn't willing to take
   it on assumption. Quorum stayed "Node and File Share Majority,"
   witness stayed Online, all three replicas stayed HEALTHY/CONNECTED
   throughout. No change to the actual error.
5. **`ccmcore.dll`, `smscore.dll`, and `tscore.dll` re-registered.**
   Selected by process of elimination: with the primary DP provider DLL
   ruled out, every other binary in the DP's `bin` folder got tested
   for self-registration, and these three were the only ones of
   ten-plus that turned out to be genuine COM servers at all (the rest
   correctly returned "no `DllRegisterServer` entry point," which is
   normal, not a finding). All three registered successfully. No
   change.

## Current status

Unresolved. Every check and fix the agent attempted from FS01's
vantage point failed to resolve it: five reasoned fixes, all ending in
failure — which led to my decision to instruct the agent to stop.
After five attempts, anything more looks like guessing.

Options on the table:

- **(Agent's Option 1)** Remove and re-add the Distribution Point role
  on FS01. Highest cost, highest confidence.
- **(Agent's Option 2)** A Process Monitor trace on FS01 to catch the
  exact failing registry lookup directly.
- **Hand the agent a WKS01 Software Center screenshot** and see
  whether it investigates via the CCM logs on its own.
- **Move the distribution point back to MECM01.** My original dissent,
  now stronger — the option I'm leaning toward.

## Lessons learned

- **AI agents can build infrastructure; that doesn't absolve the
  human.** At this point, an agent can carry an investigation start to
  finish on its own. That doesn't remove the need for the human to
  actually log into servers and verify at key milestones — not just
  review a summary.
- **A recorded dissent is worth revisiting when new evidence lands.**
  Splitting the distribution point onto FS01 was standard best
  practice, which the agent followed during the build. It's now
  affected by an unresolved content-library defect that a same-box
  MECM01 distribution point may not have had — maybe. An experienced
  MECM engineer would likely have kept the roles on the same box to
  begin with, to keep exactly this kind of issue from surfacing later.

## What's next

Both experiments play out before the next post lands: what changes
when the agent's only lead is a client-side symptom instead of a
server-side log trail, and how cleanly it manages relocating a role it
built in the first place. The next entry reports what actually
happened, not just what I intended to try.
