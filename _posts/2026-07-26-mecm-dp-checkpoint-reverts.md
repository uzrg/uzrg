---
title: Homelab Build-Out — The Distribution Point Saga, Two Checkpoint Reverts, and a Bad Diagnosis That Cost the Most Time
author: uzrg
date: 2026-07-26 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Microsoft, HyperV, Windows Server]
tags: [Microsoft, ConfigMgr, MECM, SCCM, Troubleshooting, Distribution Point, Active Directory, Checkpoints, AI Agent]
pin: false
mermaid: false
---

# Going to sleep on an open bug

The [last MECM post]({% post_url 2026-07-23-mecm-followup-fs01-distribution-point %})
left FS01's distribution point broken with an unresolved `0x80040154`
COM registration error, five clean fix attempts down, no root cause
found. Before bed I told the agent to keep going without me: try
whatever it needed to, including deleting and rebuilding the Yubico
deployment from scratch, and report back in the morning. That's a wide
grant, and a wide grant on infrastructure invites exactly the kind of
mistake this post is mostly about. Two of them were bad enough to need
a full checkpoint revert. A third wasn't a checkpoint revert at all,
but cost more time than either — a "fix" applied hours earlier in the
same session that was itself backwards, and had to be undone before
anything else could work.

**Bottom line, for anyone skimming:** the Yubico Smart Card Minidriver
is now installed on four machines through the real ConfigMgr pipeline —
WKS01, plus FS01, WSUS01, and DHCP01 added the next day. FS01 is back
to being a clean file server. MECM01 is the working distribution
point. Getting there took two Hyper-V checkpoint reverts, one config
revert of an earlier mistaken "fix," one self-inflicted IIS config
corruption caught and repaired mid-session, and — in the follow-up
work the next day — an AD schema extension that turned out to be
unnecessary and a stopped Windows service that took an unreasonable
amount of troubleshooting to explain.

## Mistake #1: picking MECM02 without checking what it was for

Early in the night, with FS01 still broken, the agent needed a working
distribution point somewhere. It picked MECM02 — a live, healthy site
server, reachable, nothing else going on. Reasonable in the moment.
Wrong given the actual plan: MECM02 is earmarked in the roadmap as the
**passive site server** for MECM01/MECM02 high availability, a role it
hasn't been asked to fill yet but is supposed to stay clean for. Giving
it distribution-point duty was scope creep into a box that has a
different job.

I caught this the next time I checked in — "why using MECM02, it was
supposed to be passive HA!" — and had it reverted rather than patched
around. The fix was a full Hyper-V checkpoint restore back to
`agent-20260724-1458-Pre-DP-role-addition`, the snapshot taken right
before any of that night's work touched the box, followed by cleanup
on the ConfigMgr side that a VM-level snapshot doesn't reach: removing
the now-stale distribution point role from the site database and
pulling MECM02 back out of the boundary group's site-system list.
Both layers had to be undone — the VM snapshot only rewinds the guest,
not the site server's own bookkeeping about it.

## Mistake #2: an IIS legacy-compatibility theory that didn't pan out

With MECM02 off the table, the agent took a second run at FS01 itself,
this time comparing its installed Windows Features against a distribution
point that worked. FS01 turned out to have `Web-Metabase` and
`Web-Mgmt-Compat` — legacy IIS 6 metabase compatibility — installed;
the working comparison box didn't. Plausible enough to test: I approved
removing them.

Checkpointed first
(`agent-20260725-1557-Pre-remove-IIS6-Metabase-compat-DP-fix-attempt`),
removed both features, rebooted. Same `0x80040154` error, immediately,
on the first retry after reboot. Theory disproven. Reverted the
checkpoint to put FS01 back exactly as it was — no half-finished state
left lying around a box that's also load-bearing for the SQL cluster's
file-share witness.

Two checkpoint reverts in one night, for two different reasons: one
because the target picked was architecturally wrong, one because a
reasonable diagnosis turned out to be incorrect. Neither is a failure
of process — checkpoints existing and getting used exactly as intended
is the process working. The point of a `Checkpoint-Lab` habit only
lands during the moment you actually need to undo something.

## Pivoting to MECM01, and finding a real, fixable defect

FS01 stayed broken with its root cause never identified. Rather than
stand up a new VM, the plan shifted to the site server itself, MECM01,
as the distribution point — something the agent had initially treated
as a dead end after an earlier session found that a distribution point
co-located with the site server "has nothing to be copied over" for
any package type. Looking again with fresh eyes turned up why: MECM01's
distribution point had, from the very start, been pointed at FS01's
**shared UNC content library**, not a local drive — matching the
original plan to centralize content on FS01, but apparently never
actually working end to end. `Move-CMContentLibrary` relocated the
site's real content library onto a local drive on MECM01, and this
time actual file content landed where it was supposed to.

That fixed one wall and immediately hit a second, worse one: both
MECM01 and MECM02 (still carrying leftover config from the earlier
detour) started returning `401 Unauthorized` to every content request
— the real ConfigMgr client, and a manual test, both. Not a permissions
problem in the normal sense: granting `Everyone: Full Control`
recursively on the content library changed nothing. Something deeper
in ConfigMgr's own content-serving stack was rejecting every request
regardless of who was asking.

## Mistake #3: reverting a "fix" that was backwards from the start

This is the one that actually cost the most time, and it wasn't a
checkpoint revert — it was undoing a decision made much earlier in
the same session and never revisited until forced to.

Hours before the 401 ever appeared, a *different* symptom — an HTTP
405 — had been diagnosed as a handler-ordering bug: ConfigMgr's own
ISAPI extension for serving content appeared to be claiming a
WebDAV-style request (`PROPFIND`) ahead of the actual WebDAV module,
so the fix at the time was to narrow that ISAPI handler's allowed verbs
and let WebDAV take PROPFIND instead. Reasonable-sounding, wrong: the
ISAPI handler is *supposed* to own PROPFIND — it's the piece that
knows how to translate a package's virtual URL into the real,
hash-addressed file on disk. Generic WebDAV has no way to do that
translation; it only understands literal filesystem paths, and
ConfigMgr's package URLs are never literal filesystem paths. Routing
PROPFIND to WebDAV didn't fix anything — it just swapped one failure
mode (405) for one that looked completely unrelated (401) and took
hours to trace back to the same line of config.

Finding it required setting up IIS Failed Request Tracing to watch
the request pipeline directly, which produced a genuine near-miss of
its own: hand-editing `applicationHost.config` to configure the trace
introduced a duplicate XML element that IIS's schema doesn't allow,
breaking the site's own configuration reads — on a box that also hosts
the Management Point for the whole ConfigMgr site, so this wasn't a
contained mistake. Caught immediately, flagged before doing anything
else, fixed with one line removed once I signed off. Confirmed the
site was fully healthy again before moving on.

With tracing actually working, the trace showed the real request
completing authentication cleanly and then getting rejected by the
WebDAV module itself, deep in request handling — confirming the
months-old "fix" was the actual cause. Reverting the handler's verb
list back to its ConfigMgr default cleared the 401 immediately, on a
direct test and then on the real client.

## What finally worked

- MECM01 as the distribution point, content library relocated to a
  local drive instead of FS01's UNC share.
- The ISAPI handler's verb list reverted to ConfigMgr's own default —
  the single fix that had been undone by mistake hours earlier.
- FS01 fully cleaned up: distribution point role removed, boundary
  group membership pulled, its leftover site-system registration
  removed from the console entirely. It's back to exactly its intended
  role — SQL Always On backups and the cluster file-share witness,
  nothing ConfigMgr-related left on it.
- MECM02 fully reverted to its clean, untouched, pre-DP-work state,
  matching its actual future role.

The Yubico Smart Card Minidriver installed on WKS01 through the real
deployment pipeline, confirmed via the client's own execution log and
the registry uninstall key for the product, not a manual workaround.

## The next day: expanding the pilot, and one more real bug

The following day's task was smaller in scope: add three more servers —
FS01, WSUS01, DHCP01 — to the pilot deployment. None had the ConfigMgr
client installed. Console-based client push got stopped by the
session's own safety controls as a real infrastructure change worth a
second look, so I handed over a client-install script of mine instead,
built around discovering the management point through Active
Directory publishing.

That AD lookup depends on the ConfigMgr site actually being published
to AD, which turned out to have never been configured. With my
approval for each step — a schema extension is not something to wave
through casually — the agent extended the AD schema, created the
publishing container, granted the site server rights to it, and
enabled publishing. The site object showed up in AD correctly. The
management point's own identity never did, even after a full service
restart. Rather than keep chasing an AD feature this environment
apparently doesn't need, the script got a straightforward edit: this
lab has exactly one site and one management point, so it's now
hardcoded rather than discovered. The schema extension is additive and
harmless to leave in place, but the feature it was meant to enable
went unused.

Two smaller bugs surfaced in the script itself under real use, both
now fixed: a missing command-line switch caused ccmsetup's own
background install to try (and fail) to rediscover the management
point through the same AD path that had just been ruled out, and the
success check trusted the wrong process exiting cleanly rather than
confirming the client service actually came up. One target server —
DHCP01 — also turned out to have its network location service
stopped, which left Windows unable to confirm real network
connectivity even though the network itself was fine; restarting the
network adapter forced a fresh check and cleared it immediately.

All three came up clean after that. The pilot collection now stands at
four machines with the Yubico Smart Card Minidriver installed through
the real pipeline: WKS01, FS01, WSUS01, DHCP01.

## Lessons learned

- **A standing "keep going while I sleep" grant needs checkpoints more
  than a supervised session does**, not less. Both real mistakes that
  night — the wrong target VM, the compatibility-feature theory that
  didn't pan out — were caught and undone cleanly because a checkpoint
  existed at the right moment, not because the agent got everything
  right the first time.
- **A VM snapshot only rewinds the guest.** Reverting MECM02 needed
  the Hyper-V checkpoint *and* a separate cleanup pass on the
  ConfigMgr site database — the two layers don't sync automatically,
  and forgetting the second one would have left stale references
  around indefinitely.
- **The most expensive mistake wasn't a VM-level one at all.** It was
  a plausible-sounding config change made hours earlier, in response
  to a different symptom, that turned a real bug into a much harder
  one to find. Config changes deserve the same "will I need to undo
  this" scrutiny as anything that gets a checkpoint — they just don't
  come with an undo button built in.
- **A confirmed symptom (0x80070005 in a Failed Request Trace) beats
  another plausible guess.** Every fix attempt before the tracing setup
  was a reasonable theory tested and discarded. The one that actually
  worked came from watching the request pipeline directly instead of
  guessing at the next layer to blame.
