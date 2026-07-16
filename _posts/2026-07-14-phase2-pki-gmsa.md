---
title: Homelab Build-Out — Phase 2, a Root CA on a Domain Controller and the gMSA Groundwork
author: uzrg
date: 2026-07-14 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Microsoft, HyperV, Windows Server]
tags: [Microsoft, Windows Server, PKI, ADCS, gMSA, Security, Automation]
pin: false
---

# Phase 2: the identity foundation

Phases 0 and 1 established what the lab actually contained and gave it
basic plumbing. Phase 2 adds identity infrastructure: a certificate
authority, certificate autoenrollment, and the group managed service
accounts (gMSAs) that later phases — RDS, Configuration Manager, SQL —
will depend on. All of this work ran on DC01, so every step was
checkpointed first and required my approval. The agent proposed; I signed
off.

## An Enterprise Root CA, knowingly on a domain controller

The CA is an Enterprise Root on DC01: `myhomelab-DC01-CA`, RSA 4096,
SHA-256, ten-year validity. Placing AD CS on a domain controller is a
documented anti-pattern in production. The CA's lifecycle becomes tied to
the DC's — you cannot rename or demote the DC without CA surgery — the
attack surface concentrates on one machine, and backup and restore of
either role now involves the other. My homelab accepts that trade-off
deliberately: one fewer always-on VM, and the DC already sits in the
availability tier a CA requires. I state it plainly because most
"CA on a DC" advice online omits the distinction that matters: acceptable
in a lab, a liability in production.

One useful discovery: an Enterprise install publishes enough default
templates that LDAPS works immediately. The Kerberos Authentication and
Domain Controller Authentication templates cover the DCs' certificate
needs without custom work.

## Template decisions, and the flag that mattered

The textbook design is one duplicated, tightly scoped template per role.
The usual tooling for that is the PSPKI module, and DC01 has no internet
access to install it. Building duplicate templates by hand means creating
raw ADSI objects with dozens of attributes that are easy to get wrong.
The agent argued that the risk outweighed the benefit in a lab, and I
agreed.

The accepted simplification: grant Domain Computers the Enroll and
Autoenroll rights on the built-in `WebServer` and `Machine` templates.
This is broader than production hygiene allows — every domain computer
can now autoenroll a web-server certificate, not only the servers that
need one. The change came to me for explicit approval before any ACL was
touched, because it is a scope decision, not a technical one.

The technical lesson from this section: the ACL alone was not enough.
`WebServer` ships with `msPKI-Certificate-Name-Flag = 1`, meaning the
subject is supplied in the request — which cannot work with
autoenrollment, where no one is present to supply a subject. The agent
changed the flag to build the subject from Active Directory, matching the
`Machine` template, and only then did enrollment succeed. If
autoenrollment silently issues nothing from a template you know is
published and permissioned, check the name flag first.

One item was deliberately left unsolved. The RDS gateway will need a
certificate carrying several subject alternative names — the internal
FQDN plus the externally published name. Autoenrollment writes only the
machine's own AD name. Rather than guess the external name now, that
certificate waits for Phase 3 and a manual `certreq` once the name is
real.

## Autoenrollment, domain-wide

A "Certificate Autoenrollment" GPO linked at the domain root enables the
machinery: enroll, renew expired certificates, and update when templates
change (`AEPolicy = 7`), verified with `Get-GPRegistryValue` rather than
by inspection. From this point, any domain-joined machine receives its
`Machine` certificate with no human involvement. That paid off two days
later, when a newly joined workstation needed to participate in the PKI
with no preparation at all — that story is the next post.

## gMSA groundwork: accounts for servers that do not exist yet

The second half of Phase 2 is the group managed service accounts that
will eventually remove static service-account passwords from SQL and
SCOM. The KDS root key went in with its effective time backdated ten
hours — the standard lab adjustment that makes the key usable immediately
instead of after the replication-safety waiting period.

Then a wrinkle worth recording: the security groups (`gMSA-SQL` for the
three SQL nodes, `gMSA-SCOM` for the SCOM server) were populated with
computer accounts for servers that are not yet built. SQL01–03 and
OPSMGR01 exist in AD only as pre-staged computer objects from the same
April batch that complicated the Phase 1 domain joins. The agent flagged
this and populated the groups anyway: the objects are valid AD
principals, and granting membership ahead of the build means the eventual
servers hold their gMSA retrieval rights from day one. The accounts
themselves (`gmsaSQL`, `gmsaSQLAgt`, `gmsaSCOM`) exist and are
retrievable by their groups.

Phase 2 did not migrate any service to a gMSA. That is a rolling,
one-node-at-a-time operation against an Always On availability group that
must stay healthy between nodes — and it requires the SQL servers to
exist first. One pre-flight check is parked for later: confirming
Microsoft's current support statement for gMSA as the Configuration
Manager site-database service account before committing the availability
group design. Building on an unsupported configuration is the kind of
mistake that surfaces at the worst possible time.

## Left open, on purpose

- **NPS01**, the eventual RADIUS server, stayed untouched. It is one of
  the machines with the unknown-local-credential problem from Phase 0,
  and the PKI work did not need to resolve that first.
- **Secrets management** was noted as a future proposal, not built. With
  a root CA in place, HashiCorp Vault becomes genuinely attractive — an
  AD secrets engine for time-bound domain credentials, a PKI engine
  issuing from this CA — but it earns its keep at Phase 5, when the lab
  has multiple secret consumers. It also must not run on the host it
  protects.

## What the agent did and what I did

The agent handled the AD CS installation and configuration, the template
ACL and name-flag changes, the GPO with registry-level verification, and
the KDS key, groups, and gMSA accounts — along with the runbook this post
condenses. I provided the approvals at every DC01 state change and made
the scope call on the template simplification. Phase 2's lesson in one
line: the difference between a lab shortcut and a mistake is whether it
was documented and approved as a shortcut.
