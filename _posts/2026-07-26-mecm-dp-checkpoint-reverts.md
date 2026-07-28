---
title: Homelab Build-Out — The Distribution Point Saga, Resolved
author: uzrg
date: 2026-07-26 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Microsoft, HyperV, Windows Server]
tags: [Microsoft, ConfigMgr, MECM, SCCM, Troubleshooting, Distribution Point, Active Directory, Checkpoints, AI Agent]
pin: false
mermaid: false
---

# Going to sleep on an unresolved failure

The [last MECM post]({% post_url 2026-07-23-mecm-followup-fs01-distribution-point %})
closed with FS01's distribution point broken by an unresolved
`0x80040154` COM registration error — five fix attempts, five failures,
no root cause found. Before bed that night I told the agent to keep
going without me: try whatever it needed, including deleting and
rebuilding the Yubico deployment from scratch if necessary, and report
back in the morning. A wide grant on live infrastructure invites the
kind of surprises this post is about. Two were bad enough to need a
full checkpoint revert — a restore to snapshot, in non-Hyper-V terms. A
third wasn't a checkpoint revert at all: an IIS configuration edit the
agent botched, then caught and diagnosed entirely on its own. The
surprising part is that fixing it required my approval, even though the
original edit that caused the mistake never needed any permission at
all.

**Bottom line:** I'm pleased with how this ended. The Yubico Smart Card
Minidriver deployed to the pilot machines once the content library and
distribution point role were relocated to MECM01 for good — WKS01 that
night, then FS01, WSUS01, and DHCP01 the following day. FS01 is back to
being a clean file server. Getting there took two checkpoint reverts,
one config mishap caught and fixed mid-session, one earlier
misdiagnosis undone before anything else could work, and — the next
day — an Active Directory schema extension that never paid off and a
stopped Windows service that took longer than it should have to
explain.

## Mistake #1: picking MECM02 without checking what it was for

Early in the night, with FS01 still broken, I handed the agent a
screenshot from WKS01: the Yubico app stuck at "Installing…," 0%
complete, going nowhere. It started where it should have — the client's
own CCM logs, not guesswork. `CAS.log` showed the same line repeating
on every retry: *"Download request only, ignoring location update."*
Confirmed: not a client problem, a missing-content problem — the same
defect the last post already knew was sitting on FS01.

It offered to keep chasing FS01's COM registration issue directly. I
gave it a different option instead: relocate the content library onto
a spare drive on MECM01 and add the distribution point role there —
get *something* working rather than keep fighting the same wall.
That's the wide grant I mentioned earlier taking its first real shape.

First snag: the agent couldn't find a second drive on MECM01 at all —
as far as it could tell, there wasn't one. I had to tell it plainly
that the drive existed before it looked properly and found it,
offline. It brought the disk online, then hit a guardrail: initializing
and formatting it through PowerShell got blocked outright, because the
permission classifier treats disk formatting as destructive — even
against a blank, never-used disk. Its way around it: `diskpart.exe`,
which isn't gated the same way.

Content relocation and the DP role on MECM01 came next, and neither
went smoothly at first. Needing a working distribution point
*somewhere* that same night, the agent picked MECM02 — a live, healthy
site server, nothing else running on it. Reasonable in the moment.
Wrong against the actual plan: MECM02 is earmarked as the **passive
site server** for MECM01/MECM02 high availability, a role it hasn't
been asked to fill yet but is supposed to stay clean for. Handing it
distribution-point duty was scope creep into a box with a different job
waiting for it.

I caught this the next time I checked in and asked — "why using
MECM02, it was supposed to be passive HA!" — and the agent offered to
reverse it rather than patch it around. The fix was a Hyper-V
checkpoint restore back to `agent-20260724-1458-Pre-DP-role-addition`,
followed by cleanup a VM snapshot doesn't reach:

```
Remove-CMDistributionPoint -SiteSystemServerName "MECM02.myhomelab.hv.lab" -SiteCode "MHL" -Force
Set-CMBoundaryGroup -Name "SUPERLAB Default Boundary Group" -RemoveSiteSystemServerName "MECM02.myhomelab.hv.lab"
```

The first strips the stale DP role out of the site database; the
second pulls MECM02 back out of the boundary group's site-system
list. Both had to be run because the snapshot only rewinds the guest,
not the site server's own bookkeeping about it.

## Mistake #2: an IIS legacy-compatibility theory that didn't pan out

With MECM02 off the table, the agent took a second run at FS01, this
time comparing its installed Windows Features against its own recall
of a distribution point that had worked. FS01 had `Web-Metabase` and
`Web-Mgmt-Compat` — legacy IIS 6 metabase compatibility — installed;
its recollection was that the working comparison box didn't.
Plausible enough to test, so when the agent asked for permission to
remove them, I approved.

Checkpointed first
(`agent-20260725-1557-Pre-remove-IIS6-Metabase-compat-DP-fix-attempt`),
removed both features, rebooted. Same `0x80040154` error, immediately.
Theory disproven. Reverted the checkpoint to put FS01 back exactly as
it had been.

Two checkpoint reverts in one night, for two different reasons: one
because the target was architecturally wrong, one because a reasonable
diagnosis was simply incorrect. Neither is a failure of process —
checkpoints existing and getting used exactly as intended *is* the
process working.

## Pivoting to MECM01, and finding a real, fixable defect

FS01 stayed broken, root cause still unidentified. Rather than stand up
a new VM, the plan shifted to the site server itself, MECM01 — not the
first time it had been tried for this, but more on that below. Looking
again with fresh eyes turned up why content had never actually copied:
MECM01's distribution point had, from the very start, been pointed at
FS01's **shared UNC content library**, not a local drive — matching the
original plan to centralize content on FS01, but apparently never
working end to end. `Move-CMContentLibrary` relocated the site's real
content library onto a local drive on MECM01, and this time actual file
content landed where it was supposed to.

That fixed one wall and immediately hit a second, worse one: both
MECM01 and MECM02 started returning `401 Unauthorized` to every content
request — the real ConfigMgr client, and a manual test, both. Not a
permissions problem in the ordinary sense: granting `Everyone: Full
Control` recursively on the content library changed nothing. Something
deeper in ConfigMgr's own content-serving stack was rejecting every
request, regardless of who was asking.

## Mistake #3: reverting a "fix" that was backwards from the start

This is the one that actually cost the most time, and it wasn't a
checkpoint revert — it was undoing a decision made much earlier in the
same session, never revisited until forced to.

Hours before the 401 appeared, a *different* symptom — an HTTP 405 —
had been diagnosed as a handler-ordering bug: ConfigMgr's own ISAPI
extension for serving content appeared to be claiming a WebDAV-style
request (`PROPFIND`) ahead of the actual WebDAV module, so the fix at
the time was to narrow that handler's allowed verbs and let WebDAV take
PROPFIND instead. Wrong: the ISAPI handler is *supposed* to own
PROPFIND — it translates a package's virtual URL into the real,
hash-addressed file on disk, something generic WebDAV can't do at all.
Routing PROPFIND to WebDAV just swapped one failure mode (405) for
another (401) that took hours to trace back to the same line of
configuration.

Finding it required IIS Failed Request Tracing, which produced a
near-miss of its own: hand-editing `applicationHost.config` to set up
the trace introduced a duplicate XML element, breaking the site's own
configuration reads — on a box that also hosts the Management Point for
the whole ConfigMgr site. The agent caught it on its own, without me
watching, and flagged it before touching anything further. Worth noting
plainly: no permission gate had stood between the agent and the edit
that caused the problem — only the *correction* needed my sign-off, an
asymmetry worth sitting with. Fixed with a single duplicate line
removed once I approved it.

With tracing working, the trace showed the real request completing
authentication cleanly and then getting rejected by the WebDAV module
itself — confirming the hours-old "fix" was the actual cause. Reverting
the handler's verb list back to ConfigMgr's default cleared the 401
immediately.

## A detour worth explaining: Package instead of Application

One more piece worth rewinding for, before wrapping up: before FS01's
metabase theory or MECM02 ever entered the picture, an earlier session
had already tried MECM01 once and walked away, because content wasn't
copying for the Yubico app at all. The distribution manager log gave a
specific-sounding reason: *"the package is a content type package.
There is nothing to be copied over."* ConfigMgr treats a modern
Application and a legacy Package + Program as different content types
internally, so maybe this only broke for Applications specifically —
worth ruling out rather than assuming away.

The Yubico Application was deleted and rebuilt as a legacy Package +
Program instead, under the standing authorization to recreate the
deployment if necessary. Same failure. Theory disproven: the bug had
nothing to do with Application versus Package. By the time the real
causes were fixed, the Package + Program version was already the object
sitting there working, so it stayed that way. All four pilot machines
run on that legacy package today; converting it to a proper Application
is on the list, just not done yet.

## What finally worked

- MECM01 as the distribution point, content library relocated to a
  local drive instead of FS01's UNC share.
- The ISAPI handler's verb list reverted to ConfigMgr's default.
- FS01 fully cleaned up: distribution point role removed, boundary
  group membership pulled, its leftover site-system registration gone
  from the console entirely. Back to exactly its intended role — SQL
  Always On backups and the cluster file-share witness.
- MECM02 fully reverted to its clean, untouched, pre-DP-work state.

<img src="{{ '/assets/img/gallery/mecm-site-system-roles-no-fs01.png' | relative_url }}" alt="Servers and Site System Roles list in the ConfigMgr console showing seven servers with no FS01 entry">
_Servers and Site System Roles: seven entries — FS01 gone from the console entirely, not just stripped of its distribution point role._

The Yubico Smart Card Minidriver installed on WKS01 through the real
deployment pipeline that same night, confirmed via the client's own
execution log and the registry uninstall key — not a manual workaround.

## The next day: expanding the pilot, and one more real bug

The following day's task was smaller: add FS01, WSUS01, and DHCP01 to
the pilot deployment. None had the ConfigMgr client installed. Console
client push got stopped by the session's safety controls as a real
infrastructure change worth a second look, so I handed over a
client-install script of mine instead, built around discovering the
management point through Active Directory publishing — the whole point
being a domain-agnostic script for installing the MECM client anywhere,
no editing required.

That assumption turned out to be false here: AD publishing had never
been configured in this domain. With my approval at each step, the
agent extended the AD schema (a genuine, necessary prerequisite —
ConfigMgr can't publish anything to AD without it), created the
publishing container, and enabled publishing. The site object showed up
in AD correctly. The management point's own identity never did, even
after a full service restart. Rather than keep chasing a feature that
wasn't paying off, the script got a straightforward edit: the site code
and the management point are both hardcoded now. That's a real step
back — the script isn't domain-agnostic anymore, it's tied to this lab
— but it's the pragmatic fix given AD publishing never delivered a
working record. The schema extension itself is harmless to leave in
place; the feature it was meant to enable just never got used.

Two smaller bugs surfaced in the script itself: a missing command-line
switch caused ccmsetup's background install to try — and fail — to
rediscover the management point through the same AD path just ruled
out, and the success check trusted the wrong process exiting cleanly
instead of confirming the client service had come up. DHCP01 also had
its network location service stopped, which left Windows unable to
confirm real connectivity even though the network was fine; restarting
the network adapter cleared it immediately.

All three came up clean after that.

<img src="{{ '/assets/img/gallery/mecm-yubico-package-deployment-success.png' | relative_url }}" alt="ConfigMgr Deployments view showing the Yubico Smart Card Minidriver Silent Install deployment at 100 percent compliance across 4 assets, 0 errors">
_Deployment status: Success 4, Error 0, 100% compliance — WKS01, FS01, WSUS01, and DHCP01 all accounted for._

## Lessons learned

- **A standing "keep going while I sleep" grant needs checkpoints more
  than a supervised session does**, not less. Both real mistakes that
  night were caught and undone cleanly because a checkpoint existed at
  the right moment, not because the agent got everything right the
  first time.
- **A VM snapshot only rewinds the guest.** Reverting MECM02 needed the
  checkpoint *and* a separate cleanup pass on the ConfigMgr site
  database — the two layers don't sync automatically.
- **The most expensive mistake wasn't a VM-level one at all.** A
  plausible-sounding config change made hours earlier, for a different
  symptom, turned a real bug into a much harder one to find. Config
  changes deserve the same "will I need to undo this" scrutiny as
  anything that gets a checkpoint — they just don't come with an undo
  button built in.
- **A confirmed symptom beats another plausible guess.** Every fix
  attempt before Failed Request Tracing was a reasonable theory, tested
  and discarded. The one that worked came from watching the request
  pipeline directly instead of guessing at the next layer to blame.

## Division of labor

The agent: every fix attempt on FS01 and MECM01/MECM02, both checkpoint
restores and the ConfigMgr-side cleanup each needed, the Failed Request
Tracing setup that isolated the 401, the FS01 teardown, the pilot
expansion script and its fixes, and the first draft of this post. Me:
the overnight go-ahead, catching the MECM02 scope mistake, approving
the metabase theory test and the schema extension step by step, and
signing off on the `applicationHost.config` repair.

## What's next

The pilot sits at four machines with a working deployment pipeline.
Converting the Yubico package into a proper Application is the obvious
next step. Beyond that, the roadmap's next real phase is the RD
Session-based farm — broker, gateway, licensing, and the session hosts
— the first build to lean on the certificate authority and the
Configuration Manager pipeline both landing cleanly before it.
