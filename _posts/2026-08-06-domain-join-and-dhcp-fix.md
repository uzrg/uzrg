---
title: Homelab Build-Out — Domain-Joining the Fleet, and the DHCP Server That Wasn't Listening
author: uzrg
date: 2026-08-06 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Microsoft, HyperV, Windows Server]
tags: [Microsoft, ConfigMgr, MECM, SCCM, Active Directory, DHCP, Checkpoints, AI Agent]
pin: false
mermaid: false
---

# Nine machines that were never actually domain members

I asked the agent to domain-join every VM that wasn't already a real member:
power on if it was off, install the OS from the template if the disk was
blank, reuse the pre-staged AD objects either way. Eight machines fit that
description — RDSCB, RDSGW, RDSLC, RDSSH01, RDSSH02 (all blank disks, never
built) plus NPS01, DEVOPS01, and OPSMGR01, which were running but turned
out to have never actually joined at all, just sitting there as pre-staged
AD reservations nobody had finished building. VMM01 stayed out of scope
that day under the standing "don't touch it yet" instruction; I brought it
in myself the next day once I actually needed it.

**Bottom line:** all nine are domain members now, running the MECM client,
carrying the Yubico driver, and enrolled in the patch ring structure from
the last post. Getting there cost more time than it should have on one
self-inflicted mistake with the OS deployment, and surfaced a second,
unrelated bug that had been sitting in the lab undetected since Phase 1:
DHCP01 had no scope at all, and even after I built one, its actual listening
socket was silently broken. Both are fixed now.

## The OOBE trap

Building a VM from the WS2025 template means copying its generalized VHDX
and letting Windows Setup run through specialize and OOBE fresh. The first
attempt, on RDSCB, mounted the copied disk before ever starting the VM and
wrote an `unattend.xml` with both the computer name and the local
Administrator password. The name didn't take, for a reason I still haven't
run down, so the agent renamed it imperatively afterward instead. The
password did take, and RDSCB joined cleanly.

From that one result, the agent concluded sysprep just carries the local
account's password over from the template's own SAM database, decided the
answer file wasn't actually necessary, and launched the other seven with no
answer file at all. All seven booted fine at the hypervisor level, heartbeat
green, genuinely running Windows, and were then completely unreachable —
"the credential is invalid," forever. They were sitting at a live,
interactive OOBE password prompt with nothing to complete it and no way for
the agent to reach a console screen and type into it.

The fix it landed on, mounting the disk again after the fact and dropping a
fresh answer file in before rebooting, looked reasonable and didn't work: it
tested that theory on one machine before trusting it across the rest, which
turned out to matter, because Windows Setup only reads an `unattend.xml`
during the genuine first specialize/OOBE pass. Once a VM has already booted
once and landed on the interactive screen, a later reboot just resumes that
same stalled session; it does not re-read anything from disk. The only real
fix was starting over completely: wipe, fresh copy from the template, write
the answer file before the disk is ever booted, exactly like RDSCB's
original sequence. That worked on all six remaining machines once it was
confirmed on one first.

## Every existing VM has a static IP, so nobody ever noticed DHCP was broken

RDSCB's first join attempt, before any of the OOBE trouble, failed for an
unrelated reason: "the specified domain either does not exist or could not
be contacted." Its adapter had an APIPA address, meaning it had never
gotten a DHCP lease in the first place. DHCP01's service was running and
AD-authorized, but had **zero scopes configured for any VLAN at all** —
matching a line item from the Phase 1 build that had apparently never
actually been finished, invisible until now because every other machine in
the lab already carries a static IP, so nothing had ever actually needed
DHCP01 to work.

I built a scope to get past it that day: `10.10.10.0/24`, dynamic range
`.100`-`.200`, everything below that excluded to protect the fifteen or so
addresses already assigned by hand. Even with a real scope active, RDSCB
still couldn't get a lease, and DHCP01's own audit log showed zero
DHCPDISCOVER entries ever, from anyone. Rather than keep chasing it that
day, the agent gave all eight machines static IPs instead, matching how
literally everything else in the lab was already configured, and moved on.

## Finding out why, the next day

I asked it to keep looking, specifically to check whether DHCP01 might be
sitting on a different subnet than the clients. It wasn't: single adapter,
correct VLAN, correct switch, matching every other machine exactly, service
binding reporting healthy. What actually cracked it was `pktmon`, capturing
UDP 67 and 68 directly on DHCP01 while the agent flipped RDSGW's adapter to
DHCP just long enough to generate one real request, then back to static
immediately after.

The broadcast arrived. It's right there in the capture, hitting DHCP01's
NIC and IP stack cleanly, and then getting dropped a few microseconds
later:

```
Drop: PktGroupId 53, Direction Rx, Type IP, DropReason INET: transport
endpoint was not found
Drop: PktGroupId 55, Direction Rx, Type IP, DropReason Port unreachable
```

Nothing was listening on UDP 67. The service said `Running`, the binding
said `True`, and neither of those was actually true at the socket level —
some stale state, never properly re-bound, probably left over from before
the scope existed at all. `Restart-Service DHCPServer -Force` fixed it in
about five seconds. A listener appeared on `10.10.10.12:67`, and the exact
same test against RDSGW that had just timed out now came back with a real
lease: `10.10.10.100`, correct subnet, correct gateway. First successful
DHCP transaction this lab has ever completed.

The eight machines from the day before kept their static IPs; nothing about
a working setup needed to change. VMM01, built the next day once I named
it explicitly, got its address the ordinary way and picked up
`10.10.10.101` without anyone assigning anything.

## Rolling out the client and the driver

With all nine actually joined, the agent ran the same
[`Install-SCCMClient.ps1`]({% post_url 2026-08-02-mecm-yubico-package-to-application %})
script from the earlier Yubico rollout against each of them, tested once on
RDSCB before trusting the rest, and confirmed all nine registered as real
MECM clients (discovery data processing lagged the actual install by a few
minutes on some of them, which is normal).

Getting each one into the right collections didn't need any new design work
because none of these nine are guarded systems: no domain controllers, no
SQL AG nodes, no MECM servers among them. They went straight into the same
two collections every other ordinary machine in the lab already uses —
`DEP - Production - Yubico Minidriver`
(the [Required deployment]({% post_url 2026-08-02-mecm-yubico-package-to-application %})
covering every non-DC domain member) and
`PATCH - Ring 2 - General Servers`
(the [auto-enabled, non-guarded ring]({% post_url 2026-08-05-wsus-patch-rings-sql-log-backups %})
that FS01, WSUS01, and DHCP01 already patch through). No new ring, no
manual gate, no collection design decisions — just direct membership rules
against structure that already existed.

Verified the driver actually landed rather than trusting the console, the
same lesson from the original Yubico post: registry ground truth
(`HKLM:\...\Uninstall\{A8C8D1E1-3BB1-470D-BBD8-3AE20FB0FD85}`) and the
client's own `CCM_Application` state both confirmed `Installed` on all
nine.

## Lessons learned

- **One working result doesn't prove the theory behind it.** RDSCB's
  password happened to come from the answer file, not from sysprep
  preserving it, and assuming the wrong mechanism cost seven machines their
  first boot.
- **An unattend.xml only gets one chance.** It has to be in place before
  the very first boot after generalize; nothing about a later reboot makes
  Windows Setup go back and re-read it.
- **"Running" and "bound" are different claims.** DHCP01's service status
  and its own binding configuration both reported healthy while nothing was
  actually listening on the port — checking the service isn't the same as
  checking the socket.
- **A config that's never been exercised can be broken for a long time
  without anyone knowing.** DHCP01 had been non-functional since Phase 1;
  it just never mattered until a machine that genuinely needed DHCP showed
  up.
- **When a live diagnostic tool exists, use it before reasoning further.**
  Every theory about DHCP01 came from comparing static configuration
  side-by-side; the actual answer took one packet capture.

## Division of labor

The agent: every deployment and rename script, isolating and fixing the
OOBE trap (including testing the fix on one machine before trusting the
batch), building the DHCP scope, the `pktmon` capture and the temporary
DHCP-client test against RDSGW, the service restart, and the full MECM
client and collection rollout across all nine machines including
verification against registry ground truth. Me: naming the original scope
of eight, deciding to work around the DHCP problem with static IPs rather
than lose more time to it that day, pointing at the subnet as worth
checking the next day, explicitly bringing VMM01 into scope afterward, and
asking for both collections on it once it was built.

## What's next

RDSCB through RDSSH02 are domain members with the client and driver
installed, but none of them carry their actual RD role yet — that's Phase
3, still ahead. NPS01, DEVOPS01, and OPSMGR01 are in the same position for
their own respective builds. VMM01 is a bare, patched, domain-joined box;
SCVMM itself is still not installed. DHCP01 being genuinely functional now
means the next machine built in this lab can just ask for an address like
anything normally would, instead of getting a static assignment by hand.
