---
title: Homelab Build-Out — YubiKey Smartcard Logon, and Read-Only pfSense Access for the Agent
author: uzrg
date: 2026-07-16 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Microsoft, HyperV, Windows Server]
tags: [Microsoft, HyperV, Windows, PKI, YubiKey, Smartcard, pfSense, Security]
pin: false
---

# Putting Phase 2's PKI to work

Phase 2 left my homelab with an Enterprise Root CA and little visibly
using it. This post covers the first real payoff: signing in to a domain
workstation with a YubiKey and a PIN instead of a password. It also
covers a prerequisite that proved valuable on its own — giving the
operations agent read-only SSH access to pfSense.

The project had three fixed boundaries, set before any work started.
First, the YubiKey is used for one purpose: authenticating users to the
domain. Second, it does not hold the CA's private key. The CA was keyed
in software during Phase 2, and moving that key into hardware would be a
separate project with its own risks. Third, administration of the
Hyper-V host itself is unchanged; the YubiKey plays no part in it. The
pilot machine is WKS01, the lab's only client workstation.

## CA side: the old v1 template, with two catches

The agent handled the CA work after I approved the DC01 changes,
checkpoint first as always. The built-in v1 Smartcard Logon template was
the right choice for a reason that matters on Windows Server 2025 domain
controllers: they enforce strong certificate mapping (the KB5014754
change, fully enforced since early 2025). A certificate enrolled through
Windows, against a template that builds its subject from Active
Directory, receives the SID security extension from the CA and maps
strongly with no extra work. The tempting alternative — generating a CSR
with ykman and submitting it offline — produces a certificate the KDC
will reject. If smartcard logon works in an older lab and fails against
2025 domain controllers, this is the most likely reason.

Two catches:

1. The template's CSP list (`pKIDefaultCSPs`) was empty, so enrollment
   would have offered no cryptographic provider. The agent set it to
   `Microsoft Base Smart Card Crypto Provider`, which is what
   minidriver-based cards, the YubiKey included, enroll through.
2. Enrollment rights went to a purpose-made **Smartcard Users** group
   rather than Domain Users. A smartcard logon certificate is a logon
   credential, and that deserves a narrower gate than the
   WebServer/Machine simplification accepted earlier in Phase 2.

## The prerequisite: read-only SSH access for the agent on pfSense

Before WKS01 could join the domain, it needed an address and working AD
DNS — and reasoning about that required knowing what pfSense was actually
doing about DHCP, an open question that had blocked Phase 1 work for
days. The rule in this lab is absolute: the agent never modifies pfSense.
Reading it, however, is fair game, so I granted read-only access.

On the pfSense side I created a local group named `Agents`
("Lab Agents - ReadAccess") containing a `LabAgent` user, with three
privileges:

- **User - System: Shell account access** — permits SSH login
- **WebCfg - All pages** — can view everything in the GUI
- **User - Config: Deny Config Write** — pfSense ignores any config.xml
  write request from this user

SSH itself is key-only (`SSHd Key Only: Public Key Only`), using an
ed25519 key whose private half lives on the host, outside version
control. pfSense is candid about this arrangement: the group page warns
that users in this group effectively have administrator-level access.
`Deny Config Write` is a guard at the PHP configuration layer, not a
security boundary — the hardening section returns to this.

Two problems surfaced during setup, both worth recording:

- **PowerShell quoting gave the SSH key an unintended passphrase.** The
  agent generated the keypair with `ssh-keygen -N '""'` from PowerShell,
  and the private key ended up protected by the literal two-character
  string `""`. The failure looked exactly like a server-side rejection
  until `ssh -v` showed pfSense accepting the key while the client failed
  to sign with it. Generate keys from a POSIX shell; PowerShell's
  native-argument quoting is unforgiving.
- **LabAgent's login shell is tcsh**, so any remote command containing
  `2>&1` fails with "Ambiguous output redirect." Wrapping remote commands
  in `sh -c '...'` resolves it.

The survey repaid the effort in a single pass. From one read of
`config.xml`: pfSense serves DHCP only on the untagged LAN, so Windows
DHCP scopes for VLANs 10–40 conflict with nothing — a Phase 1 blocker
resolved. The resolver has no domain override for the AD domain, so any
LAN client using pfSense DNS cannot locate the domain controllers — the
join would have failed without an obvious cause. And VLAN10's outbound
rule uses "OPT1 address" as its source where "OPT1 subnets" was clearly
intended, which means the domain controllers have had no outbound DNS or
NTP all along. One firewall rule edit — mine to make — fixes what Phase 0
could only flag as unexplained DNS behavior.

## WKS01: address by reservation, join by hand

I chose to pin WKS01 through a pfSense static mapping rather than a
machine-local static IP, at `.200` — the first tenant of a planned
management range (`.200`–`.254`, outside the DHCP pool) reserved for the
few systems permitted to manage pfSense. The Kea DHCP backend honored the
per-mapping DNS override, so WKS01's lease arrived pointing at the domain
controllers. I ran the join at the console; the agent verified it over
PowerShell Direct afterwards — the first time it had ever been able to
reach this machine.

The agent then installed the YubiKey Smart Card Minidriver on WKS01:
downloaded on the host (WKS01 had no internet by name — see the VLAN10
rule above), hash-verified, copied over PowerShell Direct, and installed
with `INSTALL_LEGACY_NODE=1`. That flag is Yubico's switch for smart
cards arriving over RDP redirection, which is exactly what a Hyper-V
enhanced session is.

## The lesson: where you plug in the key depends on where you sit

Hyper-V has no USB passthrough. A YubiKey reaches a VM through enhanced
session smart-card redirection — vmconnect is, in effect, an RDP client.
I plugged the key into the homelab host, opened the console, and got
"connect a smart card" no matter what I tried.

The agent worked through the host's smart-card stack — service restarts,
a ghost device left over from an earlier USB port, device cycles — all
dead ends, correctly abandoned. The event log supplied the real answer:
my "console" session on the host is itself an RDP session, because I
administer the host from my desk. Inside an RDP session, Windows
deliberately hides the host's own readers; a session sees only readers
redirected from its client. The unexpected reader that did appear in
WKS01 was my desktop PC's built-in card reader, forwarded two hops.

The fix was physical: plug the YubiKey into the machine my hands are on,
and let it ride the nested redirection chain —

```
YubiKey → my PC → mstsc (smart cards ✓) → homelab host session → vmconnect enhanced session (smart cards ✓) → WKS01
```

Nested smart-card redirection is supported, and once you know the model
it is entirely logical. Nothing needed installing anywhere except the
endpoint. Three smaller items, for completeness:

- `certutil -scinfo` against an unenrolled card reports `NTE_BAD_KEYSET`.
  That is an empty card in a working reader, not an error.
- An enhanced-session logon is an RDP logon, so the test user needed
  local **Remote Desktop Users** membership on WKS01 — granted to the
  Smartcard Users group so future pilot users inherit it.
- Enroll as the target user, not as an administrator: the template builds
  the certificate subject from the requester.

Enrollment through `certmgr.msc` wrote the key and certificate onto the
YubiKey's PIV applet, and the next sign-in was card and PIN.

## Proof, from the logs

Claims about authentication deserve log evidence. Two events, captured
minutes after the first successful card sign-in, tell the whole story.

On the domain controller that issued the ticket, Security event **4768**
(a Kerberos TGT was requested) shows the request succeeding with
certificate-based pre-authentication — `Pre-Authentication Type: 16` is
PKINIT, the smartcard path, and the certificate fields identify exactly
which credential was used:

```
Event 4768 — Kerberos authentication ticket (TGT) was requested
  Account Name:               test.labuser
  Supplied Realm Name:        MYHOMELAB.HV.LAB
  Result Code:                0x0
  Pre-Authentication Type:    16        (PKINIT — certificate/smartcard)
  Certificate Issuer Name:    myhomelab-DC01-CA
  Certificate Serial Number:  130000000F625D8FF98B14E8EE00000000000F
  Certificate Thumbprint:     DDFEFC0BC326269BE5D9A5872B75961A1DDC329B
```

A password logon would show Pre-Authentication Type 2 and no certificate
fields at all. Three seconds later, WKS01's Security log records the
matching session — event **4624**, Logon Type 10 (RemoteInteractive,
which is what an enhanced session is):

```
Event 4624 — An account was successfully logged on
  Account Name:            test.labuser
  Account Domain:          MYHOMELAB
  Logon Type:              10        (RemoteInteractive)
  Logon Process:           User32
  Authentication Package:  Negotiate (Kerberos)
```

The certificate serial number in the 4768 event matches the CA
database's issuance record for the enrollment. The chain of custody is
complete: template → enrollment → PIV applet → PKINIT → session.

## From one workstation to the whole domain

WKS01 was a pilot. For the YubiKey to authenticate its holder to any
domain-joined device, four pieces must hold everywhere — and most already
do:

1. **Chain trust — already domain-wide.** An Enterprise CA publishes its
   root certificate into Active Directory, and domain members import it
   into their Trusted Root store automatically through Group Policy
   processing; the CA also sits in the enterprise NTAuth store, which is
   what authorizes it to issue logon certificates. No per-device work.
   Verify rather than assume: `certutil -viewstore -enterprise NTAuth` on
   a member should list the CA.
2. **KDC certificates on every domain controller — already done.**
   Smartcard logon is mutual authentication: the client validates the
   DC's certificate while the DC validates the user's. Both DCs
   autoenrolled Kerberos Authentication certificates in Phase 2, and any
   future DC will do the same automatically.
3. **Card support on each target device.** Windows includes an inbox PIV
   driver that can read a YubiKey, but the Yubico minidriver is the
   supported experience — PIN management and correct multi-slot behavior.
   It is a single MSI, so the rollout mechanism is not the hard part.
   Deploying it domain-wide through Configuration Manager is a planned
   future exercise, once Phase 4 delivers a healthy MECM site; until
   then, installs are per-machine. Devices reached over RDP or enhanced
   session also need the user in their local Remote Desktop Users group —
   a group-policy item, not a PKI one.
4. **Revocation checking every participant can reach.** At each logon,
   the DC validates the user's certificate chain, including revocation
   status, and the client validates the DC's. Today the CRL is published
   at the default LDAP distribution point, which every domain member can
   reach. That is sufficient for the current fleet.

**Where OCSP comes in.** A CRL is a published list of revoked
certificates that every relying party downloads whole; OCSP replaces the
download with a signed, real-time answer about one certificate. The
practical gains are freshness — a revoked card is refused at the next
logon, not the next CRL publication — and scale. The plan here is
Microsoft's implementation, the AD CS **Online Responder**, deployed at
the proper time: when RDS and NPS make certificate validation a
high-frequency event, the responder goes on a member server, the OCSP URL
is added to the CA's AIA extension, and the CRL stays in place as the
automatic fallback. Until then, LDAP CRLs carry the load. One planning
note: AIA changes affect only newly issued certificates, so certificates
issued before the responder exists — including the pilot's — would need
re-enrollment to carry the OCSP pointer.

## Hardening next steps

The working setup is honest about its shortcuts. In priority order:

1. **Enforce read-only SSH for LabAgent with a forced command.** Today,
   "read-only" rests on `Deny Config Write` (a PHP-layer guard) and Unix
   file permissions for a non-root user. The proper enforcement is an SSH
   forced command: prefix the key in LabAgent's authorized keys with
   restriction options, so every connection can run only a vetted
   wrapper, regardless of what the client requests:

   ```
   no-port-forwarding,no-agent-forwarding,no-x11-forwarding,command="/usr/local/bin/labagent-readonly.sh" ssh-ed25519 AAAA…snip… LabAgent@readonly
   ```

   In pfSense, that whole line goes in the user's **Authorized SSH Keys**
   field in the GUI. The wrapper itself is a short shell script that
   compares `$SSH_ORIGINAL_COMMAND` — the command the client asked for —
   against an explicit allow-list and refuses everything else, including
   an empty command, which is what an interactive shell request looks
   like:

   ```sh
   #!/bin/sh
   # /usr/local/bin/labagent-readonly.sh
   # Sole entry point for LabAgent SSH sessions (forced command).
   case "$SSH_ORIGINAL_COMMAND" in
     "cat /conf/config.xml")  exec cat /conf/config.xml ;;
     "ifconfig")              exec ifconfig ;;
     "pfctl -s rules")        exec pfctl -s rules ;;
     "netstat -rn")           exec netstat -rn ;;
     "sockstat -4 -l")        exec sockstat -4 -l ;;
     *) echo "denied: ${SSH_ORIGINAL_COMMAND:-interactive shell}" >&2
        exit 1 ;;
   esac
   ```

   Exact-match patterns are deliberate — no wildcards, no argument
   pass-through — so there is nothing to inject into. Placement is an
   administrator task, consistent with the read-only rule: I copy the
   script to `/usr/local/bin/labagent-readonly.sh` over an admin SSH
   session (or paste it via **Diagnostics → Command Prompt**), owned
   `root:wheel`, mode `0555`, so LabAgent can execute it but never edit
   it. Two operational notes: pfSense upgrades can rebuild `/usr/local`,
   so the script needs re-checking after each upgrade; and the sudo
   package with a NOPASSWD allow-list is the more flexible alternative if
   the command list grows. Either way the goal is the same: the boundary
   moves from "the agent promises" to "the SSH server enforces."

2. **Restrict pfSense management to the `.200`–`.254` range** once the
   management systems all live there — today any LAN address can reach
   the GUI. Respect the anti-lockout rule and test from a `.200` machine
   before removing broader access.

3. **Fix the VLAN10 egress rule** ("OPT1 address" → "OPT1 subnets") so
   the domain controllers regain outbound DNS and NTP.

4. **PIN hygiene and enforcement**: move the PIN off the factory default,
   then set "Smart card is required for interactive logon" on the pilot
   account — which scrambles the AD password and makes the card the only
   way in.

## What the agent did and what I did

The agent handled the CA template changes, the pfSense survey and every
finding in it, the minidriver deployment, the smart-card redirection
diagnosis, the log evidence above, and the runbook this post is distilled
from. I made every pfSense change — group, user, key, static mapping —
moved the physical YubiKey, ran the domain join, performed the
enrollment, and gave the approvals in between. That division of labor is
the point: the agent never touched the firewall, and I never had to read
an event log.
