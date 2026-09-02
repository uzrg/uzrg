---
title: Homelab Build-Out — Nine Domain Joins, an OOBE Trap, and a Silent DHCP Failure
author: uzrg
date: 2026-08-06 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Microsoft, HyperV, Windows Server]
tags: [Microsoft, ConfigMgr, MECM, SCCM, Active Directory, DHCP, Checkpoints, AI Agent]
pin: false
mermaid: false
---

# Nine machines waiting for domain join

The task was to domain-join every VM that wasn't already a member:
power on if it was off, install the OS or clone from the template if
the disk was blank, reuse the pre-staged AD objects in either case.
Eight machines fit that description: RDSCB, RDSGW, RDSLC, RDSSH01,
RDSSH02 (all blank disks, never built), plus NPS01, DEVOPS01, and
OPSMGR01, which were running but turned out to have never actually
joined at all, just sitting there, pending completion. VMM01 stayed out
of scope under the standing "don't touch it yet" instruction, but was
brought in explicitly a little bit later once it was actually needed.

**Bottom line:** all nine are domain members now, have the MECM client
installed, carry the Yubico driver, and are enrolled in the patch ring
structure from the last post. Getting there took more time than it
should have, due to one self-inflicted mistake: skipping the OS
deployment's answer file on seven machines based on an untested
assumption, which left them stranded at an unreachable OOBE prompt. A
second, unrelated issue surfaced too, one that had been sitting in the
lab undetected since Phase 1: DHCP01 had no scope at all, and even
after a scope was built, its actual listening socket was silently
broken. Both are fixed now.

## The OOBE trap

Building a VM from the WS2025 template means copying its generalized
VHDX and letting Windows Setup run through specialize and OOBE fresh.
The first attempt, on RDSCB, mounted the copied disk before ever
starting the VM and wrote an `unattend.xml` with both the computer name
and the local Administrator password. The name didn't take, for a
reason still not run down, so the machine was renamed imperatively
afterward instead. The password did take, and RDSCB joined cleanly.

From that one result came an assumption that was never actually tested:
that Windows would carry the administrator password over on its own
from the template, without needing an answer file to set it. In
reality, the password worked on that first machine only because the
answer file explicitly set it, not because Windows remembered it
automatically. That distinction was never confirmed before the answer
file was dropped as "unnecessary" for the remaining seven machines,
which launched with none at all. All seven booted fine, showing green
and genuinely running Windows, and were then completely unreachable:
"the credential is invalid," forever. Each one was sitting at a live
Windows setup screen waiting for a password that was never going to be
typed in, with no way to reach a console screen and enter one by hand.

The first fix attempt, mounting the disk again after the fact and
dropping a fresh answer file in before rebooting, looked reasonable and
didn't work. Testing that theory on one machine before trusting it
across the rest turned out to matter: Windows Setup only reads an
`unattend.xml` during the genuine first specialize/OOBE pass. Once a VM
has already booted once and landed on the interactive screen, a later
reboot just resumes that same stalled session; it doesn't re-read
anything from disk. The only real fix was starting over completely:
wipe, fresh copy from the template, write the answer file before the
disk is ever booted, exactly like RDSCB's original sequence. That
worked on all six remaining machines once confirmed on one first.

## Surprise discovery: DHCP was broken

RDSCB's first join attempt, before any of the OOBE trouble, failed for
an unrelated reason: "the specified domain either does not exist or
could not be contacted." Finding it was almost an accident. Every
server in this lab normally runs on a static IP, assigned by hand as a
standard build step, but that step comes after the domain join, not
before it. A freshly built VM starts out on Windows' own default, DHCP,
until someone sets the static address, so RDSCB's very first join
attempt, before anyone had gotten to that step, was the first time in
the lab's history anything had actually needed DHCP to work. Its
adapter had an APIPA address, meaning it had never gotten a lease at
all. DHCP01's service was running and AD-authorized, but had **zero
scopes configured for any VLAN at all**, matching a line item from the
Phase 1 build that had apparently never actually been finished.

A scope went in to get past it: `10.10.10.0/24`, dynamic range
`.100`-`.200`, everything below that excluded to protect the fifteen or
so addresses already assigned by hand. Even with a real scope active,
RDSCB still couldn't get a lease, and DHCP01's own audit log showed zero
DHCPDISCOVER entries ever, from anyone. Rather than keep chasing it, all
eight machines got static IPs instead, matching how literally
everything else in the lab was already configured.

## Tracking down the DHCP issue

The next step was checking whether DHCP01 might be sitting on a
different subnet than the clients. It wasn't: single adapter, correct
VLAN, correct switch, matching every other machine exactly, service
binding reporting healthy. What actually cracked it was `pktmon`,
capturing UDP 67 and 68 directly on DHCP01:

```powershell
pktmon filter add -p 67
pktmon filter add -p 68
pktmon start --etw -m real-time
```

RDSGW's adapter flipped to DHCP just long enough to generate one real
request, then back to static immediately after.

The broadcast arrived. It's right there in the capture, hitting
DHCP01's NIC and IP stack cleanly, then getting dropped a few
microseconds later:

```text
Drop: PktGroupId 53, Direction Rx, Type IP, DropReason INET: transport endpoint was not found
Drop: PktGroupId 55, Direction Rx, Type IP, DropReason Port unreachable
```

Nothing was listening on UDP 67. The service said `Running`, the
binding said `True`, and neither of those was actually true at the
socket level: some stale state, never properly re-bound, probably left
over from before the scope existed at all. `Restart-Service DHCPServer
-Force` fixed it in about five seconds. A listener appeared on
`10.10.10.12:67`, and the exact same test against RDSGW that had just
timed out now came back with a real lease: `10.10.10.100`, correct
subnet, correct gateway. First successful DHCP transaction this lab has
ever completed.

The eight already-joined machines kept their static IPs; nothing about
a working setup needed to change. VMM01, built once it was explicitly
named, got its address the ordinary way and picked up `10.10.10.101`
without anyone assigning anything.

## Rolling out the MECM client and the Yubico driver

With all nine actually joined, the same
[`Install-SCCMClient.ps1`]({% post_url 2026-08-02-mecm-yubico-package-to-application %})
script from the earlier Yubico rollout ran against each of them, tested
once on RDSCB before trusting the rest, confirming all nine registered
as real MECM clients (discovery data processing lagged the actual
install by a few minutes on some of them, which is normal).

Getting each one into the right collections didn't need any new design
work, since none of these nine are guarded systems: no domain
controllers, no SQL AG nodes, no MECM servers among them. They went
straight into the same two collections every other ordinary machine in
the lab already uses: `DEP - Production - Yubico Minidriver` (the
[Required deployment]({% post_url 2026-08-02-mecm-yubico-package-to-application %})
covering every non-DC domain member) and `PATCH - Ring 2 - General
Servers` (the [auto-enabled, non-guarded ring]({% post_url 2026-08-05-wsus-patch-rings-sql-log-backups %})
that FS01, WSUS01, and DHCP01 already patch through). No new ring, no
manual gate, no collection design decisions, just direct membership
rules against structure that already existed.

Verified the driver actually landed rather than trusting the console,
the same lesson from the original Yubico post: registry ground truth
(`HKLM:\...\Uninstall\{A8C8D1E1-3BB1-470D-BBD8-3AE20FB0FD85}`) and the
client's own `CCM_Application` state both confirmed `Installed` on all
nine.

VMM01 is now in the same state as the other eight: domain-joined,
patched, client-installed, driver-present. SCVMM itself is still not
installed; that's a separate phase ahead.

## Lessons learned

- **One working result doesn't prove the theory behind it.** RDSCB's
  password happened to come from the answer file, not from sysprep
  preserving it, and assuming the wrong mechanism cost seven machines
  their first boot. Confirm a mechanism on a second, independent case
  before trusting it at scale.
- **An unattend.xml only gets one chance.** It has to be in place
  before the very first boot after generalize; nothing about a later
  reboot makes Windows Setup go back and re-read it. Once a machine is
  past that first boot, patching the answer file in after the fact
  won't fix it; only starting over will.
- **A fix-then-scale pattern turns a batch failure into a contained
  one.** Testing the wipe-and-rebuild fix on a single machine before
  committing the other six is what kept a bad assumption from
  compounding into a second round of failures. Validate a fix on one
  case before applying it to the rest, every time, not just after the
  first attempt already went wrong.
- **"Running" and "bound" are different claims.** DHCP01's service
  status and its own binding configuration both reported healthy while
  nothing was actually listening on the port. `Get-NetUDPEndpoint
  -LocalPort 67` would have shown the gap in seconds; check the socket,
  not just the service.
- **A config that's never been exercised can stay broken for a long
  time without anyone knowing.** DHCP01 had been non-functional since
  Phase 1; it just never mattered until a machine that genuinely needed
  DHCP showed up. Untested configuration isn't verified configuration,
  no matter how long it's been sitting there.
- **Time-boxing a troubleshooting path is a legitimate engineering
  decision, not a failure to solve it.** Switching the eight machines to
  static IPs and deferring DHCP01 kept the actual priority, getting the
  machines joined, moving forward instead of letting a secondary
  problem consume the moment. Knowing when to work around something
  instead of pushing through it is its own skill.
- **When a live diagnostic tool exists, use it before reasoning
  further.** Every theory about DHCP01 came from comparing static
  configuration side-by-side; the actual answer took one packet
  capture. Reach for the tool that shows what's actually happening on
  the wire before reasoning through what should be happening.

## Division of labor

The agent: every deployment and rename script, isolating and fixing the
OOBE trap (testing the fix on one machine before trusting the batch),
building the DHCP scope, the `pktmon` capture and the temporary
DHCP-client test against RDSGW, the service restart, and the full MECM
client and collection rollout including verification against registry
ground truth. Me: naming the original scope of eight, deciding to work
around the DHCP problem with static IPs rather than lose more time to
it, pointing at the subnet as worth checking, bringing VMM01 into scope
afterward, and asking for both collections on it once built.

## What's next

RDSCB through RDSSH02 are domain members with the client and driver
installed, but none of them carry their actual RD role yet; that's for
Phase 3, still ahead. DEVOPS01, OPSMGR01, and NPS01 are in the same
position for their own respective builds. VMM01 is a bare, patched,
domain-joined box; SCVMM itself is still not installed; that's a future
decision. DHCP01 being genuinely functional now means the next machine
built in this lab can just ask for an address like anything normally
would, instead of getting a static assignment by hand. DEVOPS01 being a
domain member now is what makes the next piece of work on it possible:
standing up an experimental development environment there.
