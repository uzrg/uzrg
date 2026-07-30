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
no root cause. Before bed that night I told the agent to keep going
without me: try whatever it needed, including rebuilding the Yubico
deployment from scratch, and report back in the morning. A wide grant
on live infrastructure invites surprises. Two were bad enough to need a
full checkpoint revert — a restore to snapshot, in non-Hyper-V terms. A
third wasn't a checkpoint revert at all: an IIS configuration edit the
agent botched, then caught and diagnosed on its own. The surprising
part: fixing it required my approval, even though the edit that caused
the mistake never needed any.

**Bottom line:** I'm pleased with how this ended. The Yubico Smart Card
Minidriver deployed to the pilot machines once the content library and
distribution point role were relocated to MECM01 for good — WKS01 that
night, then FS01, WSUS01, and DHCP01 the following day. FS01 is back to
being a clean file server. Getting there took two checkpoint reverts,
one config mishap caught and fixed mid-session, one earlier
misdiagnosis undone before anything else could work, and — the next
day — an Active Directory publishing feature written off as broken too
soon, plus a stopped Windows service that took too long to explain.

## Mistake #1: picking MECM02 without checking what it was for

Early in the night, with FS01 still broken, I handed the agent a
screenshot from WKS01: the Yubico app stuck at "Installing…," 0%
complete, going nowhere. It started where it should have — the client's
own CCM logs, not guesswork. `CAS.log` showed the same line repeating
on every retry: *"Download request only, ignoring location update."*
Confirmed: not a client problem, a missing-content problem — the same
defect the last post already knew was sitting on FS01.

It offered to keep chasing FS01's COM registration issue directly. I
redirected it instead: relocate the content library onto a spare drive
on MECM01 and add the distribution point role there — get *something*
working rather than keep fighting the same wall. That's the wide grant
I mentioned earlier taking its first real shape.

First snag: the agent couldn't find a second drive on MECM01 — as far
as it could tell, there wasn't one. I told it plainly that the drive
existed, and it found it, offline. Bringing the disk online hit
another guardrail: formatting it through PowerShell got blocked
outright, since the permission classifier treats disk formatting as
destructive, even against a blank, never-used disk. Its way around it:
`diskpart.exe`, which isn't gated the same way.

Content relocation and the DP role on MECM01 came next, neither smooth
at first. Needing a working distribution point *somewhere* that same
night, the agent picked MECM02 — a live, healthy site server with
nothing else running on it. Reasonable in the moment, wrong against the
actual plan: MECM02 is earmarked as the **passive site server** for
MECM01/MECM02 high availability, a role it's supposed to stay clean for
until asked to fill it. Handing it distribution-point duty was scope
creep into a box with a different job waiting.

I caught this the next time I checked in and asked — "why using
MECM02, it was supposed to be passive HA!" — and the agent offered to
reverse it rather than patch it around. The fix was a Hyper-V
checkpoint restore back to `agent-20260724-1458-Pre-DP-role-addition`,
followed by cleanup that a VM snapshot doesn't reach:

```
Remove-CMDistributionPoint -SiteSystemServerName "MECM02.myhomelab.hv.lab" -SiteCode "MHL" -Force
Set-CMBoundaryGroup -Name "SUPERLAB Default Boundary Group" -RemoveSiteSystemServerName "MECM02.myhomelab.hv.lab"
```

The first strips the stale DP role out of the site database; the
second pulls MECM02 back out of the boundary group's site-system list.
Both were needed because the snapshot only rewinds the guest, not
ConfigMgr's own bookkeeping about it.

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

FS01 stayed broken, root cause still unidentified. Rather than let the
agent keep reaching for one more FS01 theory, I stepped in directly:
clean the MECM footprint off FS01 entirely, full focus on MECM01
instead — not the first time MECM01 had been tried, more on that below.
With fresh eyes, the agent found why content had never actually copied
on that first attempt: MECM01's distribution point was still pointed at
FS01's **shared UNC content library**, not a local drive — matching the
original plan to centralize content on FS01, which had never actually
worked, and which the agent's own earlier attempt had missed updating.
This time, `Get-CMSite -SiteCode "MHL" | Move-CMContentLibrary
-NewLocation "E:\SCCMContentLib"` relocated the site's real content
library onto a local drive on MECM01, and actual file content landed
where it was supposed to.

That fixed one issue, but a second appeared immediately: both MECM01
and MECM02 started returning `401 Unauthorized` to every content
request — confirmed with both a real ConfigMgr client test and a
manual one. Not an ordinary permissions problem: granting `Everyone:
Full Control` recursively on the content library changed nothing.
Something deeper in ConfigMgr's own content-serving stack was rejecting
every request, regardless of who was asking — the culprit was a
misconfigured ISAPI handler left over from hours earlier in the same
session, the mistake that cost the most time of all, covered next.

## Mistake #3: a fix from hours earlier turned out to be the actual culprit

This is the one that most likely caused all the trouble. Per the
agent's own post-mortem: earlier that same night, it had been
troubleshooting a different symptom, an HTTP 405 tied to ConfigMgr's
own ISAPI extension. The "fix" at the time was narrowing that handler's
allowed verbs so WebDAV would take over a specific request type
(`PROPFIND`) instead — backwards, since the ISAPI handler is supposed
to handle that request itself. It traded one error (405) for another
(401), and the 401 took hours to trace back to the same setting.

Diagnosing it properly meant enabling IIS Failed Request Tracing. A
scripted edit to `applicationHost.config`, meant to insert one
`<traceFailedRequests />` line, inserted it twice instead — IIS's
schema only allows one, so this broke config reads outright on the
Management Point for the entire site. The agent caught this on its
own, without me watching, and asked permission to fix it. Worth noting:
nothing had gated the original risky edit, only the correction needed
my sign-off — backwards, if you stop to think about it. I approved it;
the fix was removing the duplicate line. Before, under
`system.webServer/tracing`:

```xml
<tracing>
    <traceFailedRequests />
    <traceFailedRequests />
</tracing>
```

After — back to IIS's own default, a single instance:

```xml
<tracing>
    <traceFailedRequests />
</tracing>
```

With the duplicate removed, IIS could finally generate the trace
messages needed to diagnose the 401 — the broken config had been
silently preventing tracing from producing anything useful. Once
working, it showed requests completing authentication cleanly,
confirming the handler-verb change from earlier was the real issue.
Reverting the verb list — `verb="*"`, ConfigMgr's own default, instead
of the narrowed `verb="GET,HEAD"` it had been left with — cleared the
401 immediately, and content finally made it through. That setting
lives in `applicationHost.config`, in the `system.webServer/handlers`
section scoped to the distribution point's virtual directories,
`SMS_DP_SMSPKG$` and `CCMTOKENAUTH_SMS_DP_SMSPKG$`.

**A quick side note on why this mattered so much:** every IIS request
handler is registered against a list of allowed HTTP verbs — the
request methods it will respond to, things like `GET`, `HEAD`, or
`PROPFIND` (WebDAV's method for querying file and folder metadata).
ConfigMgr's own content-serving handler needs `PROPFIND` in that list
because it's the only thing that knows how to translate a package's
virtual URL — something like `/SMS_DP_SMSPKG$/mhl00006` — into the
real, hash-addressed file sitting in the content library. Generic
WebDAV has no way to do that translation; it only understands literal
filesystem paths. Strip `PROPFIND` out of ConfigMgr's handler and
those requests fall through to WebDAV instead, which can never
resolve them — no matter what else on the site is configured
correctly.

## A detour worth explaining: Package instead of Application

One more thing worth mentioning: after the agent declared victory —
content distributed — I checked WKS01's Software Center myself and
found nothing installed. MECM01's console showed why: the Yubico
driver had been distributed as a legacy Package, not an Application,
the way it had always been set up back when FS01 was still the DP.

The answer traced back to earlier that same night, before FS01
troubleshooting had even wrapped up: the agent had seen a distribution
manager log line reading *"the package is a content type package.
There is nothing to be copied over."* Since ConfigMgr treats a modern
Application and a legacy Package + Program as different content types
internally, the agent wanted to rule out whether distribution was only
broken for Applications — so it deleted the Yubico Application and
rebuilt it as a legacy Package + Program, under the standing wide-grant
authorization to recreate the deployment if necessary.

Same failure — theory disproven, the issue had nothing to do with
Application versus Package. By the time the real causes were fixed, the
Package + Program version was already sitting there working, so it
stayed. All four pilot machines run on that legacy package today;
converting it to a proper Application is still on the list.

## What finally worked

- MECM01 as the distribution point, content library relocated to a
  local drive instead of FS01's UNC share.
- The ISAPI handler's verb list reverted to ConfigMgr's default.
- FS01 fully cleaned up: distribution point role removed, boundary
  group membership pulled, its leftover site-system registration gone
  from the console entirely. Back to exactly its intended role — SQL
  Always On backups and the cluster file-share witness.
- MECM02 fully reverted to its clean, untouched, pre-DP-work state.
- The Yubico Smart Card Minidriver installed on WKS01 through the real
  deployment pipeline.

<img src="{{ '/assets/img/gallery/mecm-site-system-roles-no-fs01.png' | relative_url }}" alt="Servers and Site System Roles list in the ConfigMgr console showing seven servers with no FS01 entry">
_Servers and Site System Roles: seven entries — FS01 gone from the console entirely, not just stripped of its distribution point role._

## The next day: expanding the pilot

The following day's task was smaller: expanding the Yubico deployment
to FS01, WSUS01, and DHCP01. None had the ConfigMgr client installed,
and console client push was stopped by the session's safety
guardrails to ask for my sign-off. Instead, I handed the agent a
client-install PowerShell script of mine — now published as
[`configmgr/Install-SCCMClient.ps1`](https://github.com/uzrg/powershell-toolkit/blob/main/configmgr/Install-SCCMClient.ps1)
in my PowerShell toolkit repo. The script is built to discover the
site code and management point via Active Directory publishing rather
than hardcoding them.

The agent extended the schema and enabled publishing in Active
Directory (both genuine prerequisites), but the objects didn't appear
right away — it takes ConfigMgr some time to complete its AD-publishing
cycle. That wasn't a show-stopper: the agent hardcoded the site code
and MP as a quick fix to keep the pilot moving. Checking again days
later, both objects were fully populated in AD, and the script's since
been reverted to real AD-based discovery.

Monitoring the deployment from the ConfigMgr console showed it
succeeding across all four targets.

<img src="{{ '/assets/img/gallery/mecm-yubico-package-deployment-success.png' | relative_url }}" alt="ConfigMgr Deployments view showing the Yubico Smart Card Minidriver Silent Install deployment at 100 percent compliance across 4 assets, 0 errors">
_Deployment status: Success 4, Error 0, 100% compliance — WKS01, FS01, WSUS01, and DHCP01 all accounted for._

## Lessons learned

- **Letting the agent work unsupervised overnight means checkpoints
  matter more, not less.** Both real mistakes that night were fixed
  cleanly because a checkpoint existed at the right moment — not
  because the agent got everything right the first time.
- **A VM snapshot only undoes changes on the VM itself.** Undoing
  MECM02's changes took two steps: restoring the checkpoint, and
  cleaning up MECM02's entry in the ConfigMgr site database. The two
  steps don't stay in sync automatically.
- **The costliest mistake wasn't a VM problem at all.** A config change
  made hours earlier, for a different problem, made a real bug much
  harder to find later. Any config change deserves the same "will I
  need to undo this?" thinking as something covered by a checkpoint —
  it just doesn't come with an automatic undo button.
- **A confirmed cause beats another guess.** Every fix tried before
  Failed Request Tracing was a reasonable idea that turned out wrong.
  The fix that actually worked came from watching the real requests
  directly, instead of guessing at what else might be wrong.
- **Permission requirements aren't symmetric.** Making the risky config
  edit needed no approval at all; fixing the mess it caused did. Worth
  keeping in mind when deciding what should actually require sign-off.
- **A guardrail on one tool doesn't block the underlying action.**
  PowerShell cmdlets like `Format-Volume` and `Remove-Item` were
  blocked as too risky, but older tools that do the same thing —
  `diskpart.exe`, `cmd /c rmdir` — weren't covered by the same
  restriction, and got used instead.

## Division of labor

The agent: every fix attempt on FS01 and MECM01/MECM02, both checkpoint
restores and the ConfigMgr-side cleanup each needed, the Failed Request
Tracing setup that isolated the 401, the FS01 teardown, the bugs it
found and fixed in the pilot expansion script, and the first draft of
this post. Me: the overnight go-ahead, catching the MECM02 scope
mistake, supplying the client-install script itself, approving the
metabase theory test and the schema extension step by step, and
signing off on the `applicationHost.config` repair.

## What's next

The pilot sits at four machines with a working deployment pipeline.
Converting the Yubico package back into a proper Application and
rolling it out to the rest of the lab is the obvious next step — along
with seeing how the agent handles staggered maintenance windows across
different deployment rings. Beyond that, MECM work pauses for now:
next up is the RD Session-based farm, then Operations Manager, Azure
DevOps, and eventually some Linux work.
