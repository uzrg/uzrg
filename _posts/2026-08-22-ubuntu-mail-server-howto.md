---
title: "Homelab Build-Out — AD-Joined Ubuntu Mail Server, with Smartcard Sign-In to Webmail"
author: uzrg
date: 2026-08-22 00:00:00 +0800
categories: [Blogging, Homelab, Virtualization, Linux, Microsoft, HyperV, Windows Server]
tags: [Linux, Ubuntu, Postfix, Dovecot, Active Directory, Kerberos, Cloud-Init, How-To, Tutorial]
pin: false
mermaid: false
---

# A step-by-step guide: Ubuntu on Hyper-V, joined to AD, running Postfix + Dovecot

Every VM in SUPERLAB up to this point has been Windows Server. This guide
walks through adding the lab's first Linux box, Ubuntu Server 24.04,
domain-joined to the same Active Directory forest as everything else,
running Postfix (SMTP) and Dovecot (IMAP) so the lab has real internal
messaging capability. The concrete driver was giving SCOM somewhere to
send alert emails, but the steps here apply just as well if you just want
a working internal mail server on a Windows-centric network.

This is written as a procedure, not a narrative; if you want the "things
that went wrong and why" story version, that's a separate post. Here,
each step includes *why* it matters, because the difference between
copy-pasting a command and understanding it is exactly what turns an
outage into a two-minute fix.

**Who this is for**: a sysadmin comfortable with basic Linux and basic
Active Directory, who hasn't necessarily joined a Linux box to AD before
or run a mail server from scratch. We'll go slower at the parts that
usually trip people up.

## What you'll end up with

- An Ubuntu Server VM, domain-joined, administered over SSH only (no
  desktop environment)
- Postfix accepting mail from specific trusted senders and delivering it
  locally
- Dovecot serving those mailboxes over IMAPS, with mailbox access gated
  by AD group membership
- A mail flow you've personally verified end-to-end, not just "the
  services are running"
- Roundcube webmail on top of that same Dovecot backend, so real users
  get a browser instead of needing an IMAP client
- Optionally (Step 9): passwordless smartcard sign-in to that webmail,
  reusing PIV certificates already issued for Windows logon

## Prerequisites

- A Hyper-V host (this guide assumes Hyper-V; the cloud-init approach
  works basically identically on any hypervisor)
- An existing Active Directory domain with at least one reachable Domain
  Controller, and a domain admin credential for the join step
- A network segment the AD DCs and DNS are reachable from
- Internet egress for the VM (for `apt`; see the firewall note in Step 3
  if your lab is normally locked down)
- About 30–45 minutes, more if this is your first time touching
  cloud-init

---

## Step 1: Decide the shape of the thing before you touch Hyper-V

Two decisions are worth making deliberately, before any VM exists:

**Name it after the OS, not the role**, unless you're certain the role is
permanent. `UBUNTU01`, not `MAIL01`. A service-facing DNS alias
(`mail.myhomelab.hv.lab`) pointing at the same IP gets you a stable
name for the *service* without locking the *hostname* to a role that
might change. If this box later also runs a wiki or a CI runner, you
won't be stuck explaining why "MAIL01" does three unrelated things.

**Decide your mail scope now**: internal-only (delivers between AD users,
no internet relay) or a real internet-facing mail server (much bigger
scope: SPF/DKIM/DMARC, reverse DNS, port 25 blocking from most consumer
ISPs and many cloud providers, spam filtering). This guide covers
**internal-only**: Postfix will only accept mail from specific trusted
senders on your network and deliver it locally. That's the right scope
for "give internal tools somewhere to send alerts" and most homelab use
cases; it sidesteps a genuinely hard problem (running a trustworthy
internet-facing MTA) that's out of scope here.

## Step 2: Build the VM with cloud-init, not the ISO installer

Clicking through the Ubuntu installer works, but it isn't repeatable and
teaches you nothing you can reuse. `cloud-init` is what real Ubuntu
deployments actually use (every Ubuntu cloud image ships with it
pre-installed), worth learning once, here, on a VM where getting it
wrong costs you nothing.

### 2a. Get the cloud image and convert it

Download the Ubuntu Server 24.04 LTS **cloud image** (not the normal
installer ISO), look for a file named something like
`ubuntu-24.04-server-cloudimg-amd64.img`. Despite the `.img` extension,
this is actually a qcow2 disk image; Hyper-V wants VHDX. `qemu-img` is
the one external tool this whole VM-build step needs, purely for that
one format conversion; everything else here (the seed ISO, `New-VM`,
Secure Boot) is built-in Windows/Hyper-V tooling with nothing extra to
install:

```powershell
# Windows/Hyper-V host: install qemu-img if you don't have it:
winget install cloudbase.qemu-img

qemu-img convert -O vhdx -o subformat=dynamic `
    ubuntu-24.04-server-cloudimg-amd64.img `
    UBUNTU01.vhdx
```

**No winget on this host?** The winget package just wraps a small,
standalone qemu-img build, not the full QEMU emulator, so you have a
few equivalent ways to get the same binary:

- Download that exact build directly:
  [cloudbase.it/qemu-img-windows](https://cloudbase.it/qemu-img-windows/),
  unzip, and run `qemu-img.exe` from wherever you extracted it, no
  installer needed.
- Chocolatey: `choco install qemu-img`. This one pulls from a
  different maintainer ([fdcastel on
  GitHub](https://github.com/fdcastel/qemu-img-windows-x64)) and
  tracks a much newer QEMU version than the winget package does.
- The full QEMU-for-Windows installer at
  [qemu.weilnetz.de/w64](https://qemu.weilnetz.de/w64/) (Stefan Weil's
  long-standing unofficial Windows builds) bundles `qemu-img.exe`
  with the entire QEMU suite, heavier (~200 MB vs. a few MB), but it's
  the actual upstream binary source the smaller packages above
  repackage.
- Already running WSL2 on this host for something else? `sudo apt
  install qemu-utils` inside it gives you Linux-native `qemu-img`; run
  the conversion against the image through its `/mnt/c/...` path and
  the resulting VHDX lands back on the Windows filesystem for Hyper-V
  to use directly.

### 2b. Build a cloud-init seed (NoCloud) ISO

Cloud-init needs three files, `meta-data`, `user-data`, and
`network-config`, packaged into an ISO with the volume label `cidata`.
This ISO gets attached as a virtual DVD; cloud-init reads it on first
boot and never needs it again after that.

**`meta-data`** just needs an instance ID (bump this on any *later*
seed rebuild to force cloud-init to reprocess everything; otherwise it
assumes it's already configured this instance and skips re-running):

```yaml
instance-id: ubuntu01-v1
local-hostname: UBUNTU01
```

**`network-config`** matches the NIC by driver, not by name. Hyper-V's
synthetic network adapter doesn't get a predictable interface name you
can guess in advance:

```yaml
version: 2
ethernets:
  eth0:
    match:
      driver: hv_netvsc
    set-name: eth0
    dhcp4: false
    addresses: [10.10.10.34/24]
    routes:
      - to: default
        via: 10.10.10.1
    nameservers:
      addresses: [10.10.10.10, 10.10.10.11]
      search: [myhomelab.hv.lab]
```

**`user-data`** sets the initial admin account, and *critically*, the
firewall. Get the SSH source scoping right here or you'll lock yourself
out on first boot:

```yaml
#cloud-config
hostname: UBUNTU01
fqdn: UBUNTU01.myhomelab.hv.lab
users:
  - name: labadmin
    groups: sudo
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... your-public-key-here
    sudo: ['ALL=(ALL) NOPASSWD:ALL']

package_upgrade: true
packages:
  - linux-tools-virtual
  - linux-cloud-tools-virtual

runcmd:
  - ufw allow from 10.10.10.0/24 to any port 22 proto tcp
  # if you administer this VM from a DIFFERENT subnet than the VM's own
  # subnet (very likely, your management host probably isn't ON the
  # server VLAN), add that subnet too. Check with a route lookup on the
  # host you'll SSH from BEFORE you assume "same subnet as the VM":
  #   Find-NetRoute -RemoteIPAddress <VM's IP>   (PowerShell)
  #   ip route get <VM's IP>                      (Linux)
  - ufw allow from 10.10.50.0/24 to any port 22 proto tcp
  - ufw --force enable
```

**Two package gotchas worth knowing before you hit them:**

- Install `linux-tools-virtual` / `linux-cloud-tools-virtual` for Hyper-V
  integration (IP reporting in Hyper-V Manager, etc.), **not**
  `qemu-guest-agent`, which is the equivalent tool for a *different*
  hypervisor (QEMU/KVM) and does nothing useful here. Heartbeat and time
  sync work out of the box regardless (in-kernel `hv_utils` driver), so
  you'll only notice this one if Hyper-V Manager isn't showing an IP.
- If `package_upgrade: true` pulls in a new kernel version while these
  Hyper-V tool packages are also installing, the tool packages can end up
  built against a kernel that isn't the one currently running yet. If
  `hv-kvp-daemon` won't start after first boot, a plain `sudo reboot`
  almost always fixes it; the currently-running kernel and the
  currently-installed kernel-tools packages just need to match.

Build the actual ISO. On Linux/macOS this is one line:

```bash
genisoimage -output seed.iso -volid cidata -joliet -rock meta-data user-data network-config
```

On Windows, there's no built-in equivalent. PowerShell's naive
`[IStream]$comObject` cast against the `IMAPI2FS` COM object doesn't work
(the interface QI fails through PowerShell's late-binding). The reliable
path is a small inline C# helper via `Add-Type` that does the COM cast
properly from C#, then a manual read/write loop to burn the files onto
the image. It's more code than you'd expect for "make an ISO," but it's
a one-time script you can keep and reuse:

```powershell
param(
    [Parameter(Mandatory)] [string]$SourceDir,
    [Parameter(Mandatory)] [string]$OutputIso,
    [string]$VolumeLabel = 'cidata'
)

Add-Type -TypeDefinition @'
using System;
using System.IO;
using System.Runtime.InteropServices.ComTypes;

public static class IsoStreamHelper
{
    public static void SaveToFile(object streamObj, string path)
    {
        IStream stream = (IStream)streamObj;
        byte[] buffer = new byte[65536];
        IntPtr bytesReadPtr = System.Runtime.InteropServices.Marshal.AllocHGlobal(4);
        using (FileStream fs = new FileStream(path, FileMode.Create, FileAccess.Write))
        {
            while (true)
            {
                stream.Read(buffer, buffer.Length, bytesReadPtr);
                int bytesRead = System.Runtime.InteropServices.Marshal.ReadInt32(bytesReadPtr);
                if (bytesRead <= 0) break;
                fs.Write(buffer, 0, bytesRead);
                if (bytesRead < buffer.Length) break;
            }
        }
        System.Runtime.InteropServices.Marshal.FreeHGlobal(bytesReadPtr);
    }
}
'@

$fsi = New-Object -ComObject IMAPI2FS.MsftFileSystemImage
$fsi.FileSystemsToCreate = 3        # ISO9660 + Joliet, matches genisoimage's -joliet -rock intent
$fsi.VolumeName = $VolumeLabel

$root = $fsi.Root
$root.AddTree($SourceDir, $false)   # $false: add the folder's contents directly, don't nest under its name

$resultImage = $fsi.CreateResultImage()
[IsoStreamHelper]::SaveToFile($resultImage.ImageStream, $OutputIso)

Write-Host "Wrote $OutputIso"
```

The `IStream` interface is the actual fix here: C# can call its `Read`
method correctly because it implements the interface directly, where
PowerShell's dynamic dispatch through the same COM object fails the
interface query. The 4-byte allocation for `bytesReadPtr` isn't
arbitrary either: native `IStream::Read` writes a 32-bit `ULONG` into
that pointer, not a 64-bit value, so reading it back with `ReadInt32`
(not `ReadInt64`) has to match; get this mismatched and the read count
comes back as garbage from whatever the extra 4 bytes happened to
contain, intermittently rather than every time, which makes it a nasty
one to debug if you ever go looking for "why does my ISO sometimes
build fine and sometimes throw a range exception." `$SourceDir` should
contain *only* the three seed
files, nothing else; `AddTree` with `$false` adds everything inside
that folder straight into the ISO's root, not nested under a
subdirectory named after it. Run it against your three seed files:

```powershell
.\New-CloudInitSeedIso.ps1 -SourceDir .\seed-files -OutputIso .\seed.iso
```

**Verify it before trusting it**, the same way you'd verify anything
else in this guide: mount the result and read the files back, don't
just check that the script exited without an error.

```powershell
$mount = Mount-DiskImage -ImagePath .\seed.iso -PassThru
$vol = $mount | Get-Volume
"Volume label: $($vol.FileSystemLabel)"   # expect: cidata
Get-ChildItem "$($vol.DriveLetter):\"      # expect exactly the 3 seed files, nothing else
Dismount-DiskImage -ImagePath .\seed.iso
```

### 2c. Create the VM

```powershell
New-VM -Name UBUNTU01 -Generation 2 -MemoryStartupBytes 2GB `
    -VHDPath .\UBUNTU01.vhdx -SwitchName "Internal-VSwitch-LAN-TRUNK"
Set-VMNetworkAdapterVlan -VMName UBUNTU01 -Access -VlanId 10
Set-VMMemory UBUNTU01 -DynamicMemoryEnabled $true -MinimumBytes 512MB -MaximumBytes 4GB
Set-VMFirmware UBUNTU01 -EnableSecureBoot On -SecureBootTemplate MicrosoftUEFICertificateAuthority
Add-VMDvdDrive UBUNTU01 -Path .\seed.iso
Start-VM UBUNTU01
```

`Internal-VSwitch-LAN-TRUNK` is exactly what its name says, a trunk
carrying every VLAN through to pfSense for inter-VLAN routing, so a VM
attached to it with no VLAN configured gets untagged traffic, not
whichever VLAN you assumed. `Set-VMNetworkAdapterVlan -Access -VlanId
10` is what actually puts this NIC on VLAN 10 (servers) to match the
`10.10.10.0/24` address configured in `network-config` above; skip it
and the VM boots fine but can't reach anything on that subnet, in a way
that looks like a cloud-init networking failure rather than the VLAN
mismatch it actually is.

The Secure Boot template matters: Generation 2 VMs default to the
**Windows** certificate authority template, which won't boot a Linux
guest. `MicrosoftUEFICertificateAuthority` is the one that works for
Ubuntu's signed shim/GRUB chain.

Give it a minute or two, then confirm you can SSH in:

```bash
ssh -i your_key labadmin@10.10.10.34
```

If SSH times out but `ping` works, that's almost always the firewall
source-scoping gotcha from 2b: `ufw`'s default rules permit ICMP
regardless of the default-deny policy, so ping succeeding tells you
nothing about whether your actual admin traffic is allowed through.

## Step 3: Join to Active Directory

Install the join tooling and attempt the join:

```bash
sudo apt update
sudo apt install -y realmd sssd sssd-tools libnss-sss libpam-sss adcli samba-common-bin

sudo realm join --user=administrator myhomelab.hv.lab
```

**If you get "Message stream modified"** (this is
`KRB5KRB_AP_ERR_MODIFIED`, a GSS-API checksum/signing mismatch; it is
**not** clock skew, which gives a distinctly different "clock skew too
great" error), that's a known `realmd`/`adcli` interop issue against some
Windows Server versions. Don't chase permissions or NTP, just retry with
the alternate backend:

```bash
sudo realm join --membership-software=samba --user=administrator myhomelab.hv.lab
```

This uses `net ads join` (from `samba-common-bin`) instead of `adcli`
under the hood, and in practice just works where `adcli` doesn't. It'll
happily pick up and finish a partial computer-account join that `adcli`
left behind, no manual cleanup needed in AD first.

**Scope who's actually allowed to log in.** By default, `sssd`'s
`access_provider = ad` lets *every* domain account log into this box,
almost certainly not what you want. Create an AD security group for
exactly who should have access, and restrict to it:

```bash
sudo tee -a /etc/sssd/sssd.conf > /dev/null <<'EOF'
access_provider = simple
simple_allow_groups = SUPERLAB-MailUsers@myhomelab.hv.lab
EOF
sudo systemctl restart sssd
```

Verify the restriction *without* a real login attempt, using
`sssctl`'s built-in check:

```bash
sudo sssctl user-checks -a acct test.labuser@myhomelab.hv.lab
```

Look for `pam_acct_mgmt: Success` for a group member and
`pam_acct_mgmt: Permission denied` for someone who isn't; confirms the
group scoping is actually doing something before you rely on it.

## Step 4: Get a TLS certificate from your internal CA

If you're running an internal AD Certificate Services CA (common in a
lab like this), and your Linux box isn't itself domain-joined to
something that can reach the CA API directly, the simplest path is:
generate the CSR on the Linux box, submit it from a domain-joined
Windows machine that *can* reach the CA.

```bash
# On the Linux VM:
openssl req -new -newkey rsa:2048 -nodes \
    -keyout mail01.key -out mail01.csr \
    -subj "/CN=mail.myhomelab.hv.lab" \
    -addext "subjectAltName=DNS:mail.myhomelab.hv.lab"
```

Copy `mail01.csr` to a Windows machine that can reach your CA, then:

```powershell
certreq -submit -config "DC01.myhomelab.hv.lab\myhomelab-DC01-CA" `
    -attrib "CertificateTemplate:WebServerManualSAN" mail01.csr mail01.cer
```

(Use whichever certificate template your CA has that allows a subject
name not tied to the requesting computer's own AD identity; a plain
`WebServer` template usually needs SAN specified via `-attrib` as shown.
`WebServerManualSAN` above is exactly that, a custom template on this
lab's CA for this case, the same one already used for OPSMGR01's web
console cert; your CA admin may need to set up an equivalent if a
stock template doesn't cover it.)

Export your CA's own root certificate too (`Cert:\LocalMachine\Root` in
the Windows certificate store), copy both back to the Linux VM, and
combine issued-cert + key into a full-chain PEM Postfix and Dovecot can
both reference:

```bash
sudo mkdir -p /etc/ssl/mail01
sudo cp mail01.cer mail01.key ca-root.cer /etc/ssl/mail01/
cat /etc/ssl/mail01/mail01.cer /etc/ssl/mail01/ca-root.cer \
    | sudo tee /etc/ssl/mail01/mail01-fullchain.pem > /dev/null
```

## Step 5: Install and configure Postfix

```bash
sudo apt install -y postfix
```

During install, Debian/Ubuntu's Postfix package asks a couple of
debconf questions (mail server type, "system mail name"). Pick
**Internet Site**, even though Step 1 scoped this box to internal-only
mail: the label describes Postfix's delivery style (direct SMTP, no
smarthost), not whether it's internet-facing, and it's the only option
that sets `mydestination` to your actual domain rather than just
`localhost`. For the system mail name, enter your domain
(`myhomelab.hv.lab`); that value is what lands in `mydestination`,
which is what makes Postfix accept mail addressed to
`@myhomelab.hv.lab` as local delivery instead of something to relay
elsewhere.

Edit `/etc/postfix/main.cf`:

```
# Only accept mail from specific trusted senders - NOT the whole internet,
# NOT even your whole internal network unless you actually want that.
# Scope this to a subnet, not a single host - see the note right after this.
mynetworks = 127.0.0.0/8, 10.10.10.0/24

# Force IPv4 only if you don't have IPv6 routing sorted out - avoids a
# class of confusing connectivity issues where Postfix tries IPv6 first,
# fails, and the failure mode isn't obviously "IPv6 problem" from the log.
inet_protocols = ipv4

smtpd_tls_cert_file = /etc/ssl/mail01/mail01-fullchain.pem
smtpd_tls_key_file = /etc/ssl/mail01/mail01.key
smtpd_tls_security_level = may
```

`mynetworks` isn't just descriptive shorthand here, it's the literal
Postfix parameter you just set in `main.cf` above: a list of IP ranges
Postfix treats as fully trusted, exempt from the relay checks applied
to every other client. That's exactly why it's the single most
important line in this whole config to get right. For an internal-only
server like this one, `mynetworks` isn't one layer of access control
among several, it effectively *is* the access control for "who can
send mail through this server." Scope it to exactly the hosts/subnets
that need to relay through it, not "the whole LAN because it's
easier."

```bash
sudo systemctl restart postfix
sudo ufw allow from 10.10.10.0/24 to any port 25 proto tcp   # match your mynetworks scope
```

**The trap: scoping this to one host instead of one subnet.** It's
tempting to lock `mynetworks` and the matching `ufw` rule to the exact IP
of whatever machine happens to be sending mail today; it looks tighter,
and it's exactly what broke a real deployment of this guide. That
server was receiving alert notifications from a clustered monitoring
system whose notification workflow doesn't run from one fixed server;
it runs from whichever node in a management-server pool currently owns
that role, and ownership moves. A rule scoped to a single host works
fine right up until the workload moves to a different node in the same
pool, and then mail delivery fails with a firewall-level connection
timeout that looks nothing like a firewall problem from the sending
side: the client just times out, because the server never saw the
connection at all. Scope both `mynetworks` and the matching firewall
rule to the actual subnet (or subnets) that legitimately needs to send
mail, sourced from your network's own authoritative record: AD Sites
and Services subnets in a Windows-integrated environment, your IPAM or
VLAN documentation otherwise, not to whichever single IP happens to be
sending mail on the day you write the rule.

**`ufw` and `mynetworks` don't have to match, and sometimes shouldn't.**
They answer two different questions, enforced at two different layers.
`ufw` asks first: can this connection reach port 25 at all. `mynetworks`,
further downstream, asks a narrower one of a connection that already got
past `ufw`: is this client allowed to relay to a destination this server
*isn't* authoritative for, meaning anything outside `mydestination`.
That second question is enforced specifically by `defer_unauth_destination`
in `smtpd_relay_restrictions`, and it only ever fires for destinations
outside `mydestination`. Delivery to a local mailbox never counts as
"unauthorized" in that sense; `mydestination` already covers it. So for
mail to a local mailbox, `ufw` alone decides who gets in; `mynetworks`
is never even consulted. `mynetworks` only governs the other case:
whether that same client can also use this server to relay mail out to
somewhere else. That means it's entirely legitimate for `ufw` to admit a
subnet `mynetworks` doesn't: a workstation subnet, say, whose apps need
to submit alerts to a mailbox here but should never be trusted to relay
this server out to the internet on their behalf. If you deliberately
leave a subnet out of `mynetworks` like this, write down *why* right
next to the setting. The alternative is a future admin finding the gap,
assuming it's an oversight, and "fixing" it by widening `mynetworks` to
match `ufw`, quietly turning every host on that subnet into a trusted
relay client the day someone adds a restriction (an RBL check, a rate
limit, a SASL requirement) that leans on `permit_mynetworks` as its
bypass.

## Step 6: Install and configure Dovecot

```bash
sudo apt install -y dovecot-core dovecot-imapd dovecot-lmtpd
```

**The single biggest gotcha in this whole guide** is mail storage
location. The obvious-looking config is:

```
mail_location = maildir:~/Maildir
```

**Don't use this.** It works for a real interactive login (SSH, IMAP)
because PAM's `pam_mkhomedir` module creates the home directory (and
Dovecot then creates `Maildir` inside it) on first real session, but
LMTP mail delivery from Postfix is *not* a real interactive session, so
`pam_mkhomedir` never fires for it. The very first piece of mail you try
to deliver will fail with something like:

```
mkdir(/home/test.labuser@myhomelab.hv.lab/Maildir) failed: Permission denied
```

Use a dedicated mail store instead, independent of OS home directories;
this is also just the more standard pattern for AD/LDAP-backed Dovecot
setups generally, not a workaround:

```bash
sudo mkdir -p /var/mail/vmail
sudo chown root:mail /var/mail/vmail
sudo chmod 3777 /var/mail/vmail
```

(Mode `3777`, world-writable with both the sticky bit and setgid set,
looks alarming but is correct here. The sticky bit (the leading `1`)
means each user's own delivery process, which runs as *that user's*
real uid/gid via Dovecot's userdb, can create its own subdirectory
without needing write access to anyone else's, and can't delete
another user's directory either. The setgid bit (the leading `2`,
`1+2=3` combined) is the one that's easy to skip and breaks things
subtly if you do: every subdirectory a delivery process creates under
here inherits the parent's group, `mail`, instead of whatever primary
group that specific AD user happens to have. That matters because
`mail_privileged_group = mail` below depends on every mailbox actually
being group `mail`; skip setgid and mailboxes created by different
users end up with inconsistent group ownership, and that setting stops
working reliably.)

`/etc/dovecot/conf.d/10-mail.conf`:

```
mail_location = maildir:/var/mail/vmail/%u/Maildir
mail_privileged_group = mail
```

`/etc/dovecot/conf.d/10-ssl.conf`:

```
ssl = required
ssl_cert = </etc/ssl/mail01/mail01-fullchain.pem
ssl_key = </etc/ssl/mail01/mail01.key
```

Now wire Dovecot's LMTP service to actually receive mail from Postfix.
In `/etc/dovecot/conf.d/10-master.conf`, find the `service lmtp` block
and point its unix listener at where Postfix expects to find it:

```
service lmtp {
  unix_listener /var/spool/postfix/private/dovecot-lmtp {
    mode = 0600
    user = postfix
    group = postfix
  }
}
```

Then tell Postfix to actually use it, back in `/etc/postfix/main.cf`:

```
mailbox_transport = lmtp:unix:private/dovecot-lmtp
```

```bash
sudo systemctl restart dovecot postfix
sudo ufw allow from 10.10.10.0/24 to any port 993 proto tcp
sudo ufw allow from 10.10.50.0/24 to any port 993 proto tcp   # or wherever you administer/connect from
sudo ufw allow from 10.10.10.0/24 to any port 143 proto tcp
sudo ufw allow from 10.10.50.0/24 to any port 143 proto tcp
```

Dovecot listens on both 993 (implicit TLS) and 143 (STARTTLS) by
default. With `ssl = required` set above, a plaintext session on 143 is
rejected until it upgrades to TLS, so both ports are equally safe to
expose, open both deliberately rather than leaving 143 reachable and
unaudited just because nothing happened to connect to it yet.

## Step 7: Verify the whole chain, for real

Don't stop at "the services started." Send an actual message from
another machine and retrieve it over IMAP; that's the only test that
proves Postfix, LMTP handoff, Dovecot delivery, and IMAPS auth are all
actually wired together correctly:

`Send-MailMessage` is the obvious cmdlet here, and it's deprecated:
Microsoft's own docs warn it can't negotiate TLS securely and recommend
against using it, pointing at `System.Net.Mail.SmtpClient` instead. But
that class carries the same warning in its own docs now, pointing at a
third-party library (MailKit) instead, and pulling in a NuGet package
for one throwaway test message is a lot of new surface for what this
step actually needs. So talk raw SMTP instead, the same way this
step's IMAP check further down talks raw IMAP: no deprecated API,
nothing to install, and you can see exactly what's crossing the wire.

```powershell
# from a Windows box permitted in your mynetworks scope:

# The obvious, deprecated one-liner, left here for reference only, not
# because it's wrong for a quick lab test, but because it's the first
# thing anyone reaches for and worth knowing why this guide doesn't:
# Send-MailMessage -To test.labuser@myhomelab.hv.lab -From test@myhomelab.hv.lab `
#     -Subject "test" -Body "test" -SmtpServer mail.myhomelab.hv.lab -Port 25

$client = [System.Net.Sockets.TcpClient]::new('mail.myhomelab.hv.lab', 25)
$stream = $client.GetStream()
$reader = [System.IO.StreamReader]::new($stream)
$writer = [System.IO.StreamWriter]::new($stream)
$writer.NewLine = "`r`n"
$writer.AutoFlush = $true

function Send-Line ($line) { $writer.WriteLine($line); Write-Host ">> $line" }
# EHLO's reply is multiple lines, "250-...", ending in a final "250 ...";
# every other command replies with exactly one line.
function Read-Reply { do { $r = $reader.ReadLine(); Write-Host "<< $r" } while ($r -match '^\d{3}-'); $r }

Read-Reply | Out-Null                                   # 220 greeting
Send-Line "EHLO $env:COMPUTERNAME";                     Read-Reply | Out-Null
Send-Line "MAIL FROM:<test@myhomelab.hv.lab>";          Read-Reply | Out-Null
Send-Line "RCPT TO:<test.labuser@myhomelab.hv.lab>";    Read-Reply | Out-Null
Send-Line "DATA";                                       Read-Reply | Out-Null
Send-Line "Subject: test"
Send-Line ""
Send-Line "test"
Send-Line "."
Read-Reply | Out-Null
Send-Line "QUIT";                                       Read-Reply | Out-Null

$writer.Close(); $reader.Close(); $client.Close()
```

```bash
# check delivery landed:
sudo ls /var/mail/vmail/test.labuser@myhomelab.hv.lab/Maildir/new/

# check the mail log for a clean send:
sudo grep 'test.labuser@myhomelab.hv.lab' /var/log/mail.log | tail -5
# look for "status=sent" - "status=deferred" means something (usually the
# mail_location bug above, if you skipped that step) is still wrong
```

Then confirm you can actually retrieve it: any IMAP client pointed at
`mail.myhomelab.hv.lab:993` (IMAPS) with the AD credential for that
mailbox should show the message. If you don't have a GUI mail client
handy, `openssl s_client -connect mail.myhomelab.hv.lab:993 -quiet`
followed by manually typing IMAP commands (`a1 LOGIN user pass`, `a2
SELECT INBOX`, `a3 FETCH 1 BODY[]`) will do it, though a real client is
obviously less painful.

At this point you have a working internal mail server. Everything below
is optional, and the second half (smartcard login) is a genuinely
advanced, fragile addition; read the warning at the top of Step 9 before
committing to it.

## Step 8: Add webmail: Roundcube

IMAP clients are fine for admins; most real users want a browser.
Roundcube is a lightweight, well-documented webmail client that talks to
the same Dovecot backend you already built.

### 8a. Install the stack

```bash
sudo apt install -y nginx php-fpm php-mbstring php-xml \
    roundcube-core roundcube-sqlite3
```

**`roundcube-imap` isn't a real package**, and if you type it into this
command anyway (easy to assume from the pattern of `roundcube-core` +
`roundcube-sqlite3`), `apt install` fails outright on the unknown name
and aborts the *entire* line, nginx and php-fpm included, not just the
one bad package. Roundcube's IMAP support lives inside `roundcube-core`
itself, not a separate package. `php-imap` and `php-curl` are real,
installable packages, but neither is actually needed here and neither
ends up installed by this exact command: Roundcube's own bundled IMAP
client (`rcube_imap_generic.php`, see Step 9d) doesn't use PHP's `ext-imap`
at all, and nothing in this guide's plugin code calls cURL. Both are
common inclusions in generic "install Roundcube" tutorials that assume
a broader use case than this one.

**Real gotcha**: on Debian/Ubuntu, `roundcube-core` pulls in Apache2 as a
Recommends *regardless of what you actually want to run*. If you're using
nginx (as here), that's an unwanted second web server fighting nginx for
port 80. Purge it right after install:

```bash
sudo apt purge -y apache2 apache2-bin apache2-data apache2-utils libapache2-mod-php8.3
```

The Debian package wires up Roundcube's own preferences/cache database
via `dbconfig-common` automatically (SQLite here, for a low-traffic
homelab install), no manual DB setup needed.

### 8b. Configure Roundcube

`/etc/roundcube/config.inc.php`, the two settings that matter most:

```php
$config['imap_host'] = ['ssl://mail.myhomelab.hv.lab:993'];
$config['smtp_host'] = 'localhost:25';
```

These two intentionally point at **different addresses**, for opposite
reasons:

- **IMAP uses the real hostname**, not `127.0.0.1`: your Dovecot TLS
  cert was issued for `mail.myhomelab.hv.lab` (Step 4), and PHP's TLS
  stream verifies the hostname against the certificate. `127.0.0.1` would
  fail that check even though the connection itself would otherwise work
  fine.
- **SMTP uses plain loopback**, not the real hostname: Postfix's
  `mynetworks` (Step 5) only permits specific trusted senders, not this
  box's own address. Loopback traffic bypasses `ufw` entirely and is
  already covered by the `127.0.0.0/8` entry already sitting in
  `mynetworks` from Step 5, so it "just works" without widening your
  relay scope.

### 8c. nginx vhost

```nginx
server {
    listen 80;
    server_name mail.myhomelab.hv.lab;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name mail.myhomelab.hv.lab;

    ssl_certificate     /etc/ssl/mail01/mail01-fullchain.pem;
    ssl_certificate_key /etc/ssl/mail01/mail01.key;

    root /var/lib/roundcube/public_html;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }
}
```

(nginx 1.24, which ships with Ubuntu 24.04, needs `http2` combined into
the `listen` line as shown above; the separate `http2 on;` directive
some newer examples use requires nginx 1.25.1+.)

### 8d. The certificate-trust gotcha

The very first login attempt will likely fail with something like:

```
Could not connect to ssl://mail.myhomelab.hv.lab:993: Unknown reason
```

This isn't an authentication failure; it's a **TLS trust failure**.
PHP's SSL stream wrapper verifies Dovecot's certificate against the
box's *system* CA trust store. Windows domain clients get your internal
CA's root trusted automatically via AD/GPO; a fresh Linux box never gets
that unless you add it yourself:

```bash
sudo cp /etc/ssl/mail01/ca-root.cer /usr/local/share/ca-certificates/myhomelab-dc01-ca.crt
sudo update-ca-certificates
```

Unlike every other cert filename in this guide, the destination name
here is cosmetic, not load-bearing. `update-ca-certificates` (see its
man page) only requires two things: PEM format, and a `.crt` extension
on anything under `/usr/local/share/ca-certificates/`; it doesn't care
what the base name is, it just globs the directory. `ca-root.cer`
itself is untouched either way, since `cp` only ever writes the
destination; name it whatever's clear to you, `myhomelab-dc01-ca.crt`
here just documents which CA it came from. And since nothing exists at
that destination path yet on a fresh box, this specific run *creates*
the file rather than overwriting it, though running the same command
again later would overwrite it with whatever's currently in
`ca-root.cer`.

(Don't "fix" this by disabling TLS verification; that defeats the point
of using TLS at all. Add the real trust anchor instead.)

```bash
sudo ufw allow from 10.10.10.0/24 to any port 443 proto tcp
sudo ufw allow from 10.10.50.0/24 to any port 443 proto tcp
```

**One port this vhost never gets a `ufw` rule for: 80.** That's on
purpose, but make it a decision, not an oversight. The `listen 80`
block above only exists to return a redirect, and a service *listening*
on a port isn't the same as that port being *reachable*: with no
firewall rule, the redirect never runs, and a bare `http://` request
just hangs instead of bouncing cleanly to HTTPS. Two legitimate choices
here: open port 80 to the same subnets as 443 for a clean redirect
experience, or leave it closed so an `http://` attempt fails outright at
the network layer instead of ever reaching the redirect. For a mail
server carrying credentials and message content, closing it is the
safer default; a redirect-only listener that *can* be reached just
gives plaintext HTTP a foothold before it gets turned away. This guide
leaves it closed; state your own choice explicitly rather than letting
it default by omission.

### 8e. Verify

Log into `https://mail.myhomelab.hv.lab/` with a real AD credential
from a mailbox you've already sent test mail to (Step 7). Confirm you
can see that message; that proves nginx, PHP-FPM, Roundcube's IMAP
client, and Dovecot are all correctly wired together, not just that the
login page loaded.

Then audit the firewall as a whole, not just the port you were just
working on:

```bash
# What's actually listening:
sudo ss -tlnp

# What ufw actually permits:
sudo ufw status verbose
```

Compare the two lists by hand. Every listening port should show up in
your firewall rules deliberately, either allowed from a specific
subnet for a reason you can state, or intentionally left closed. A port
that's merely listening-but-unexamined, open by omission rather than by
decision, is exactly the kind of gap that stays invisible until the day
something on a different subnet actually needs it.

---

## Step 9: Optional: certificate-based (YubiKey/smartcard) login for Roundcube

**Read this before starting.** This is a substantially bigger, more
fragile undertaking than everything above; it touches Active Directory
delegation (a real security-sensitive change, not something to do without
sign-off if you're not the only admin), requires building a custom
Kerberos bridge script, and has several genuinely non-obvious integration
bugs between how Kerberos represents identities as strings and how mail
servers expect usernames to look. If you just want webmail, stop at Step
8; this section is for adding *passwordless smartcard login* to that
webmail, reusing PIV/smartcard certificates you've already issued for
Windows logon.

**Prerequisites**: your AD already has PKI set up with a certificate
template that issues both **Client Authentication** and **Smart Card
Logon** EKUs (the same template used for Windows smartcard logon), and
target users already hold a certificate from it. This guide doesn't cover
issuing that certificate, only wiring Roundcube to accept it.

### Why Kerberos Constrained Delegation, not a simpler proxy

Two real designs solve "a validated browser certificate becomes an IMAP
login":

- **Dovecot master-user proxy**: one dedicated "master" account with its
  own password can authenticate *as* any real user. Simpler, no AD
  changes needed, but introduces one new local secret capable of
  authenticating as any mailbox user (mitigated by nginx only reaching
  that code path for someone who already holds a valid, non-revoked
  client cert, but still a new secret sitting on disk).
- **Kerberos Constrained Delegation with Protocol Transition
  (S4U2Self/S4U2Proxy)**: the mail server's own machine account gets AD
  delegation rights to obtain a real, genuinely passwordless Kerberos
  ticket on the user's behalf. No new secret anywhere. This is what's
  documented below, architecturally the "correct" match for what IIS
  does natively for smartcard-authenticated web apps on Windows, but with
  real added complexity since Linux has no built-in equivalent to
  Windows' AD certificate-mapping.

Before the config and code below, here's the whole path a login actually
takes, since it crosses five separate pieces and it's easy to lose track
of which one does what:

```
Browser                    presents PIV cert (TLS client cert)
│
└── nginx                  verifies cert against CA, passes verdict to PHP
    │
    └── Roundcube plugin   reads verified UPN from the cert's SAN
        │
        └── s4u_bridge.py  runs as root, sudo'd via one narrow rule
            │
            ├── kinit as machine account          ──►  AD / KDC
            ├── S4U2Self + S4U2Proxy               ──►  AD / KDC
            └── IMAP service ticket back           ◄──  AD / KDC
            │
            └── writes ccache to tmpfs
                │
                └── Roundcube plugin   hands ccache path to Dovecot (GSSAPI)
                    │
                    └── Dovecot        validates ticket, opens the mailbox
```

Five things have to each do their own job correctly for this to work:
nginx only *validates* the certificate and hands off a verdict, it
never touches Kerberos at all; the plugin never talks to AD directly,
it only reads the already-verified identity and calls the bridge
script; the bridge script is the only piece that ever touches the
machine account's keytab or performs the actual S4U exchange; and
Dovecot only ever sees a finished ticket in a ccache file, it has no
idea a certificate was involved at all. Each of Steps 9a-9g below
builds one link in this chain; if the smartcard login fails end to
end, work through them in this same left-to-right order rather than
guessing which piece is broken.

### 9a. AD side: register the SPN and grant delegation

This is the one step in this whole guide that's a genuine security
decision, not just a config change; get explicit sign-off if you're not
the sole admin of this AD forest before running it. From a domain
controller or any machine with the AD PowerShell module:

```powershell
setspn -A imap/mail.myhomelab.hv.lab UBUNTU01

Set-ADComputer -Identity "UBUNTU01" -Add @{
    'msDS-AllowedToDelegateTo' = @(
        'imap/mail.myhomelab.hv.lab'
        'imap/mail.myhomelab.hv.lab@MYHOMELAB.HV.LAB'
    )
}

# NOTE: computer accounts need the SamAccountName's trailing '$' here -
# Set-ADAccountControl's -Identity does NOT resolve a bare computer name
# the same way Set-ADComputer does; this is a real, easy-to-hit mistake.
Set-ADAccountControl -Identity "UBUNTU01$" -TrustedToAuthForDelegation $true
```

What this actually grants, precisely: `UBUNTU01$` can obtain a Kerberos
ticket for the `imap/mail.myhomelab.hv.lab` service **on behalf of any
AD user**, without needing that user's password. It is *not* scoped to a
specific user at the AD layer, only to this one SPN. Which user
actually gets impersonated is entirely enforced by your application code
(the nginx cert check + Roundcube plugin below), not by AD.

### 9b. Get a Dovecot keytab for that SPN, without touching the machine's password

You need Dovecot to hold a Kerberos key for `imap/mail.myhomelab.hv.lab`
so it can validate tickets. The tempting tools here (`net ads keytab`,
`msktutil`) both turn out to be dead ends on a `realmd`/`sssd`-joined
box: `net ads keytab` needs Samba's own local secrets store, which a
`realmd` join never populates, and `msktutil` can only add a *new* SPN's
key by either knowing the machine account's current plaintext password
(you don't have it) or rotating it (risky, that's the same identity your
SSH/sssd/mail-delivery already depend on).

The actual fix: since the SPN lives on the box's *own* computer account,
its Kerberos key is byte-identical to the key already sitting in
`/etc/krb5.keytab` under other aliases (`host/UBUNTU01`, `UBUNTU01$`,
etc). Dump those raw key bytes and build a new keytab entry under the
`imap/...` principal, reusing them exactly: no AD change, no password
rotation.

`klist`, `kinit`, and `kvno` (used throughout the rest of this section)
all come from `krb5-user`, not from anything `realmd`/`sssd` already
pulled in for the domain join:

```bash
sudo apt install -y krb5-user
```

```bash
sudo klist -k -e -K -t /etc/krb5.keytab
# note the aes256/aes128 hex key values shown for any UBUNTU01$ entry
```

`ktutil`'s interactive `addent -key` can't be scripted (its hex prompt
reads the controlling terminal directly, like a password prompt; it
silently accepts nothing piped in non-interactively). Build the keytab
file's bytes directly instead, with a short, fully non-interactive Python
script using only the standard library:

```python
import struct, time

def entry_bytes(components, realm, name_type, kvno, enctype, key_hex):
    key = bytes.fromhex(key_hex)
    body = struct.pack('>H', len(components))
    body += struct.pack('>H', len(realm)) + realm.encode()
    for c in components:
        body += struct.pack('>H', len(c)) + c.encode()
    body += struct.pack('>i', name_type)
    body += struct.pack('>I', int(time.time()))
    body += struct.pack('>B', kvno & 0xff)
    body += struct.pack('>H', enctype)
    body += struct.pack('>H', len(key)) + key
    body += struct.pack('>I', kvno)
    return body

realm = 'MYHOMELAB.HV.LAB'
components = ['imap', 'mail.myhomelab.hv.lab']
kvno = 3  # match whatever kvno klist showed above
entries = [
    (18, 'PASTE_AES256_HEX_HERE'),   # aes256-cts-hmac-sha1-96
    (17, 'PASTE_AES128_HEX_HERE'),   # aes128-cts-hmac-sha1-96
]

with open('/tmp/dovecot.keytab.new', 'wb') as f:
    f.write(struct.pack('>H', 0x0502))
    for enctype, key_hex in entries:
        body = entry_bytes(components, realm, 1, kvno, enctype, key_hex)
        f.write(struct.pack('>i', len(body)))
        f.write(body)
```

Verify the output's key bytes match the source **exactly** before
installing (`klist -k -e -K -t /tmp/dovecot.keytab.new`), then:

```bash
sudo mv /tmp/dovecot.keytab.new /etc/dovecot/dovecot.keytab
sudo chown root:dovecot /etc/dovecot/dovecot.keytab
sudo chmod 640 /etc/dovecot/dovecot.keytab
```

### 9c. Enable GSSAPI in Dovecot: two real gotchas here

```bash
sudo apt install -y dovecot-gssapi
```

**Gotcha #1**: `dovecot-gssapi` is a separate package from
`dovecot-core`. Enabling `auth_mechanisms = gssapi` without it installed
doesn't fail gracefully; it crashes Dovecot's *entire* `auth` process
(`Fatal: Unknown authentication mechanism 'GSSAPI'`), taking down **all**
logins including plain password auth, not just GSSAPI. Install the
package first.

`/etc/dovecot/conf.d/10-auth.conf`:

```
auth_krb5_keytab = /etc/dovecot/dovecot.keytab
auth_mechanisms = plain gssapi
auth_username_format = %n@myhomelab.hv.lab
```

**Gotcha #2, the trickiest one in this whole section**: that
`auth_username_format` line isn't optional. When you later do S4U2Self
using AD's "enterprise name" form of a UPN (which contains its own
embedded `@`), the resulting Kerberos client principal comes back as
something like `test.labuser\@myhomelab.hv.lab@MYHOMELAB.HV.LAB`, a
literal backslash escaping the UPN's own `@`, standard Kerberos string
serialization for a name that contains that character. Dovecot's
`auth_username_chars` rejects that raw backslash outright, and even if
you widen the character whitelist, `sssd` still can't resolve a username
containing one. The fix used in the bridge script below sidesteps the
escaping (uses a bare local-part principal instead of the full UPN, so
there's nothing to escape), but that alone leaves Dovecot deriving just
the *bare* username with the Kerberos realm stripped off, which `sssd`
(if it requires fully-qualified names, check `use_fully_qualified_names`
in `sssd.conf`) still won't resolve. `auth_username_format` reconstructs
the fully-qualified form from the bare name Dovecot actually derives.

### 9d. Build php-krb5

Not in Ubuntu's apt repos, build from source via PECL:

```bash
sudo apt install -y libkrb5-dev php8.3-dev build-essential php-pear
sudo pecl install krb5
echo "extension=krb5.so" | sudo tee /etc/php/8.3/mods-available/krb5.ini
sudo phpenmod krb5
sudo systemctl restart php8.3-fpm
```

Roundcube's own IMAP client (`rcube_imap_generic.php`) already has
built-in GSSAPI support once this extension is present, no core patches
needed, just a populated Kerberos credential cache handed to it at login
time, which is exactly what the bridge script below produces.

### 9e. The S4U2Self/S4U2Proxy bridge script

This is the piece with no off-the-shelf tool: something has to
authenticate as the machine account, impersonate the target user
(S4U2Self, no password needed; that's exactly what the delegation grant
in 9a allows), and exchange that for a real service ticket to the IMAP
SPN (S4U2Proxy), then hand the result to Dovecot as a Kerberos
credential cache file.

An initial attempt built this directly against `python-gssapi`'s raw
S4U2Self/S4U2Proxy bindings; it got both delegation steps working but
every attempt to export the resulting ticket to an on-disk ccache failed
with an opaque low-level error, regardless of which part of that API was
used. MIT krb5's own `kvno` command-line tool, built specifically for
testing S4U delegation, does the exact same sequence correctly on the
first try, so the final version shells out to `kinit`/`kvno` instead of
fighting the Python bindings:

`/opt/roundcube-smartcard/s4u_bridge.py` (root:root, mode `0750`):

```python
#!/usr/bin/env python3
import sys
import os
import subprocess
import tempfile

IMAP_SPN = 'imap/mail.myhomelab.hv.lab@MYHOMELAB.HV.LAB'
MACHINE_KEYTAB = '/etc/krb5.keytab'
MACHINE_PRINCIPAL = 'UBUNTU01$@MYHOMELAB.HV.LAB'
WWW_DATA_USER = 'www-data'


def run(cmd, **kwargs):
    return subprocess.run(cmd, check=True, capture_output=True, text=True, **kwargs)


def main():
    if len(sys.argv) != 3:
        print('usage: s4u_bridge.py <target-upn> <output-ccache-path>', file=sys.stderr)
        return 2

    target_upn = sys.argv[1]
    ccache_path = sys.argv[2]

    machine_ccache = tempfile.NamedTemporaryFile(prefix='s4u_machine_', suffix='.ccache', delete=False)
    machine_ccache.close()

    try:
        # Get a TGT for the machine account from its own keytab - the
        # identity TrustedToAuthForDelegation was granted to.
        run(['kinit', '-k', '-t', MACHINE_KEYTAB, '-c', machine_ccache.name, MACHINE_PRINCIPAL])

        # S4U2Self (impersonate target_upn, no password needed) + S4U2Proxy
        # (-P, constrained delegation to the IMAP SPN) in one step.
        #
        # Use -I with just the bare local part, NOT -U with the full UPN:
        # -U treats the string as a Kerberos "enterprise name", which for
        # a UPN containing its own embedded "@" produces an escaped,
        # unusable client principal (see the auth_username_format note
        # above). The bare local part has nothing to escape.
        target_local_part = target_upn.split('@', 1)[0]
        env = dict(os.environ, KRB5CCNAME=f'FILE:{machine_ccache.name}')
        run(
            ['kvno', '-I', target_local_part, '-P', '--out-cache', ccache_path, IMAP_SPN],
            env=env,
        )

        # kvno creates ccache_path as root (this script runs as root via a
        # narrow sudoers rule); hand it to www-data, the only other reader
        # that needs it.
        run(['chown', f'{WWW_DATA_USER}:{WWW_DATA_USER}', ccache_path])
        os.chmod(ccache_path, 0o640)

        print(f'OK: ccache written to {ccache_path} for {target_upn}')
        return 0

    except subprocess.CalledProcessError as exc:
        print(f'S4U bridge failed for {target_upn}: {exc.cmd} -> {exc.stderr}', file=sys.stderr)
        return 1

    finally:
        os.unlink(machine_ccache.name)


if __name__ == '__main__':
    sys.exit(main())
```

Narrow, purpose-built sudo access so the web server can invoke this one
script as root (needed for the machine keytab), and nothing else:

```bash
echo 'www-data ALL=(root) NOPASSWD: /opt/roundcube-smartcard/s4u_bridge.py' \
    | sudo tee /etc/sudoers.d/roundcube-smartcard
sudo chmod 0440 /etc/sudoers.d/roundcube-smartcard
sudo visudo -c
```

The bridge writes each ticket into `/run/roundcube-smartcard/`, tmpfs,
memory-backed, wiped on reboot. `/run` only allows root to create *new*
subdirectories, so declare it via `systemd-tmpfiles` rather than letting
the app try to `mkdir` it itself at runtime (which would silently fail):

```bash
echo 'd /run/roundcube-smartcard 0750 www-data www-data -' \
    | sudo tee /etc/tmpfiles.d/roundcube-smartcard.conf
sudo systemd-tmpfiles --create /etc/tmpfiles.d/roundcube-smartcard.conf
```

### 9f. nginx: request the client certificate, but make it optional

```nginx
ssl_client_certificate /etc/ssl/mail01/ca-root.cer;
ssl_verify_client optional;   # "optional" is load-bearing - keeps the
                               # plain password form working for everyone
                               # without a smartcard
ssl_verify_depth 2;
```

In the PHP location block, pass the result through to PHP-FPM:

```nginx
fastcgi_param SSL_CLIENT_VERIFY $ssl_client_verify;
fastcgi_param SSL_CLIENT_CERT $ssl_client_escaped_cert;
```

**These are two different nginx variables, deliberately.**
`$ssl_client_verify` is nginx's own verification result, the literal
string `SUCCESS`, `NONE` (no certificate presented), or a specific
failure reason. That's what the plugin's `get_verified_upn()` checks
against `=== 'SUCCESS'` further down; passing the certificate itself
into this field, even by copy-paste mistake, silently breaks the whole
feature; the string comparison never matches, so the smartcard option
never even appears, with nothing in any log to point at why.
`$ssl_client_escaped_cert` is the actual certificate content, used
specifically for `SSL_CLIENT_CERT`, not the older `$ssl_client_cert`;
nginx's `$ssl_client_cert` embeds the certificate's line breaks as
literal `\t` continuations (a legacy header-compatibility format),
which PHP's `openssl_x509_parse()` silently fails to parse. The escaped
variant is standard URL-encoding, which the plugin below
`urldecode()`s back into real PEM before parsing.

### 9g. The Roundcube plugin

Two real architectural lessons went into this plugin's final shape,
worth understanding before you copy it:

- **A custom `register_action()` handler is never reachable before
  login.** Roundcube's core `index.php` only special-cases
  `task=login&action=login` (and `action=oauth`) prior to
  authentication; any other registered action just silently
  re-renders the plain login page, no matter what URL points at it. The
  smartcard button below links to the *same* `login` action with an
  extra marker instead, and an `authenticate` hook (the same pattern the
  bundled `autologon` plugin uses) does the real work.
- **GSSAPI logins have no password to fall back on for reconnects.**
  Roundcube's session-reconnect logic (used for every page load/AJAX
  request after the first) re-authenticates using a password read back
  out of the session, which is empty for a GSSAPI login. The fix is to
  mark the session as smartcard-authenticated and re-derive a **fresh**
  ticket via `storage_init()` on *every* connect attempt, not just the
  first.

Worth being clear about what that second point actually means
operationally: `s4u_bridge.py` is not a setup-time script that runs
once and hands off a long-lived credential. It's invoked fresh on
*every single* `storage_init` call, meaning every page load and every
AJAX request for as long as that smartcard session is active, and each
call does a brand-new `kinit` from the machine account's keytab plus a
full S4U2Self/S4U2Proxy exchange against the KDC, not a reuse of
anything from the call before it. At homelab scale that's invisible;
on a busier deployment with many concurrent smartcard sessions, that's
real, continuous load against the domain controllers that scales with
page views, not with logins. A future improvement worth exploring would be caching a ticket for its
actual remaining lifetime (a plain `klist` against the ccache already
shows the expiry) and only re-deriving once it's genuinely close to expiring,
rather than on every single request. This guide doesn't implement
that; it's a real gap worth knowing about before sizing this setup for
anything beyond a handful of lab mailboxes.

`/var/lib/roundcube/plugins/smartcard_login/smartcard_login.php`, note
this path specifically: on Debian's Roundcube package, the *live* plugin
directory is `/var/lib/roundcube/plugins/`, not `/usr/share/roundcube/plugins/`
(the latter holds only the stock packaged plugins, symlinked into the
former):

```php
<?php
class smartcard_login extends rcube_plugin
{
    // Deliberately no $task restriction here - see the callout right
    // after this code block before you add one back in.

    private $bridge_script = '/opt/roundcube-smartcard/s4u_bridge.py';
    private $ccache_dir = '/run/roundcube-smartcard';

    function init()
    {
        $this->add_hook('template_object_loginform', array($this, 'inject_smartcard_button'));
        $this->add_hook('authenticate', array($this, 'authenticate'));
        $this->add_hook('storage_init', array($this, 'storage_init'));
    }

    private function get_verified_upn()
    {
        if (empty($_SERVER['SSL_CLIENT_VERIFY']) || $_SERVER['SSL_CLIENT_VERIFY'] !== 'SUCCESS') {
            return null;
        }
        if (empty($_SERVER['SSL_CLIENT_CERT'])) {
            return null;
        }

        $pem = urldecode($_SERVER['SSL_CLIENT_CERT']);
        $cert = openssl_x509_parse($pem);
        if ($cert === false || empty($cert['extensions']['subjectAltName'])) {
            return null;
        }

        // subjectAltName looks like: 'othername:UPN:test.labuser@myhomelab.hv.lab'
        if (preg_match('/([A-Za-z0-9._-]+@[A-Za-z0-9.-]+)/', $cert['extensions']['subjectAltName'], $m)) {
            return $m[1];
        }
        return null;
    }

    function inject_smartcard_button($args)
    {
        $upn = $this->get_verified_upn();
        if ($upn === null) {
            return $args;
        }

        $url = rcmail::get_instance()->url(array(
            '_task' => 'login',
            '_action' => 'login',
            '_smartcard' => 1,
        ));
        $safe_upn = htmlspecialchars($upn, ENT_QUOTES);

        $button = '<div class="smartcard-login-option" style="margin-top:1em;text-align:center;">'
                 . '<a href="' . $url . '" class="button mainaction">'
                 . 'Sign in with smartcard as ' . $safe_upn
                 . '</a></div>';

        $args['content'] = $args['content'] . $button;
        return $args;
    }

    function authenticate($args)
    {
        if (empty($_GET['_smartcard']) && empty($_POST['_smartcard'])) {
            return $args;
        }

        $upn = $this->get_verified_upn();
        if ($upn === null) {
            $args['valid'] = false;
            $args['error'] = 'Smartcard certificate not presented or not valid.';
            return $args;
        }

        // The actual ticket is obtained fresh in storage_init() below, on
        // this request and every later one - not here.
        $_SESSION['smartcard_upn'] = $upn;

        $args['user'] = $upn;
        $args['pass'] = '';
        $args['host'] = rcmail::get_instance()->autoselect_host();
        $args['valid'] = true;
        $args['cookiecheck'] = false;

        return $args;
    }

    function storage_init($args)
    {
        if (empty($_SESSION['smartcard_upn'])) {
            return $args;
        }

        $ccache_path = $this->obtain_ccache($_SESSION['smartcard_upn']);
        if ($ccache_path === null) {
            return $args;
        }

        if (!empty($_SESSION['smartcard_ccache_cleanup']) && $_SESSION['smartcard_ccache_cleanup'] !== $ccache_path) {
            @unlink($_SESSION['smartcard_ccache_cleanup']);
        }
        $_SESSION['smartcard_ccache_cleanup'] = $ccache_path;

        $args['auth_type'] = 'GSSAPI';
        $args['gssapi_cn'] = 'FILE:' . $ccache_path;
        $args['gssapi_context'] = 'imap/mail.myhomelab.hv.lab';
        return $args;
    }

    private function obtain_ccache($upn)
    {
        if (!is_dir($this->ccache_dir)) {
            @mkdir($this->ccache_dir, 0750, true);
        }
        $ccache_path = $this->ccache_dir . '/rc-' . bin2hex(random_bytes(16)) . '.ccache';

        $cmd = sprintf(
            'sudo %s %s %s 2>&1',
            escapeshellarg($this->bridge_script),
            escapeshellarg($upn),
            escapeshellarg($ccache_path)
        );
        exec($cmd, $output, $exit_code);

        if ($exit_code !== 0) {
            rcube::write_log('smartcard_login', 'S4U bridge failed for ' . $upn . ': ' . implode(' ', $output));
            return null;
        }

        return $ccache_path;
    }
}
```

**Gotcha #3, and the nastiest one in this whole guide: don't add
`public $task = 'login';` to this class.** It looks like the obviously
correct thing to set on a login plugin, and it's wrong. Roundcube's
plugin loader treats a non-empty `$task` as a filter on the *entire*
class, not just its login-specific parts: every hook the class
registers, including `storage_init`, gets silently skipped unless the
current request's task matches that string. Ordinary mail browsing runs
under the `mail` task, not `login`, so the one login connection that
happens during sign-in still works fine, and every connection after it,
for the rest of the session, silently falls back to Roundcube's
password-based reconnect, fails client-side before a byte reaches
Dovecot, and surfaces as `Server Error: Empty password` with a
permanently empty inbox. The trap: Dovecot's own log shows a completely
clean, successful GSSAPI login the whole time, because the one
connection that does succeed is the internal one Roundcube's core uses
to validate the credential during login itself. If a smartcard sign-in
works but nothing ever loads afterward, and every server-side log looks
healthy, check this property before anything else.

Two more real bugs worth calling out explicitly if you deviate from this:
`$this->api->url()` does **not** exist on Roundcube's plugin API; the
correct call is `rcmail::get_instance()->url(...)`, as used above.
Setting `$rcmail->config->set('imap_gssapi_cn', ...)` also does
**nothing**: Roundcube's `storage_init()` only reads a fixed set of
`{$driver}_*` config keys into its options array; GSSAPI parameters have
to be injected through the `storage_init` hook itself, exactly as this
plugin does.

Enable it in `/etc/roundcube/config.inc.php`:

```php
$config['plugins'] = ['smartcard_login'];
```

### 9h. Verify end-to-end

Standalone-test the bridge script first, through the exact path
Roundcube uses (`sudo -u www-data sudo ...`), before touching a browser:

```bash
sudo -u www-data sudo /opt/roundcube-smartcard/s4u_bridge.py \
    test.labuser@myhomelab.hv.lab /tmp/test.ccache
sudo klist -e -f -c /tmp/test.ccache
sudo rm -f /tmp/test.ccache
```

(`-e` shows the encryption type actually negotiated, `-f` shows the
ticket's flags; `-c` is just being explicit about credential-cache
mode, which is `klist`'s default anyway. It doesn't take
`/tmp/test.ccache` as its own argument the way `-t` takes a keytab name
in 9b's `klist` calls; the path here is parsed as the plain trailing
argument regardless of whether `-c` is present.)

You should see a valid ticket for `imap/mail.myhomelab.hv.lab`. Only
once that works standalone, test the real flow: insert the YubiKey,
browse to `https://mail.myhomelab.hv.lab/`, confirm the "Sign in with
smartcard as ..." button appears, click it, enter the PIN, and confirm
you land in the inbox; then reload the inbox and open a message, to
confirm the *reconnect* path (9g's `storage_init` re-derivation) works
too, not just the initial login. This lab's actual run of that sequence:

![SUPERLAB Webmail login page with the smartcard sign-in option]({{ '/assets/img/gallery/ubuntu01-roundcube-smartcard-login-button.png' | relative_url }})
_The login page's "Sign in with smartcard as ..." link, generated by `inject_smartcard_button()`_

![Browser certificate selection dialog]({{ '/assets/img/gallery/ubuntu01-roundcube-smartcard-cert-selection.png' | relative_url }})
_Browser prompts for which certificate to present: this is the client cert nginx will verify_

![Windows Security smart card PIN prompt]({{ '/assets/img/gallery/ubuntu01-roundcube-smartcard-pin-prompt.png' | relative_url }})
_PIN prompt for the smartcard itself, before the certificate is released to the browser_

![Signed-in inbox showing real mail]({{ '/assets/img/gallery/ubuntu01-roundcube-smartcard-login-inbox.png' | relative_url }})
_Landed in the inbox with no password ever entered, and, post-Gotcha-#3 fix, still populated after a reload_

Regression-test plain password login after every change in this
section: it should keep working unchanged throughout, since nothing
here should ever be able to break the path a browser with no client
certificate takes.

## What's next

That's the whole build: an AD-joined Ubuntu mail server, webmail on top
of it, and (if you took Step 9) passwordless smartcard login for that
webmail reusing existing PIV infrastructure. From here, the natural
extensions are the same ones any mail server eventually needs: mailbox
quotas, Sieve filtering, calendaring. None of these are specific to the
AD-integration work this guide focused on.

Before any of that, though, there's a more foundational piece worth
doing first: centralized event log collection. Right now Postfix,
Dovecot, Roundcube, and sssd/realmd's auth events all just sit in local
files on one Ubuntu VM. That's fine while it's a one-person lab, but the
moment more than one stakeholder has a reason to care about this box,
security wanting failed-auth attempts, whoever owns compliance wanting
a record of who accessed which mailbox, you six months from now trying
to figure out why a delivery silently stopped, scattered local logs
stop being enough. The same applies to every other server and piece of
supporting infrastructure in this lab, not just this one. Wiring the
lab into a central log collector, ideally built as prep for a SIEM
rather than added as an afterthought, pays for itself before any of
the extensions above do.

There's also a maintenance and monitoring angle worth covering on its
own: certificate renewal for the mail01/nginx certs, keytab rotation for
the S4U bridge, and SCOM coverage for Postfix/Dovecot/Roundcube health.
That's a big enough topic to earn its own follow-up rather than a rushed
paragraph here.

One thing that follow-up should probably tackle head-on: Step 9's S4U
bridge currently re-derives a ticket from scratch on *every* IMAP
connection, not just at login (see the callout in 9g). That's fine at
this lab's scale, but it's real, continuous KDC load that grows with
page views rather than logins, and it's the kind of design decision
that deserves a proper look, likely ticket caching keyed on remaining
lifetime, before this pattern gets reused anywhere with more than a
handful of concurrent smartcard sessions.

## Further reading

All the pieces this build covers have their own documentation worth
knowing where to find:

- [Ubuntu Server Guide](https://ubuntu.com/server/docs/): general
  Ubuntu Server administration, including its own
  [SSSD with Active Directory how-to](https://ubuntu.com/server/docs/how-to/sssd/with-active-directory/),
  covering the same domain-join territory as Step 3.
- [Postfix Documentation](https://www.postfix.org/documentation.html):
  the canonical reference for every `main.cf` parameter this guide
  touches, `mynetworks` and `mydestination` included.
- [Dovecot Documentation](https://doc.dovecot.org/): covers
  `mail_location`, GSSAPI authentication, and the LMTP service in far
  more depth than Steps 6 and 9c go into.
- [Roundcube Documentation](https://docs.roundcube.net/): the plugin
  API Step 9g's `smartcard_login` plugin builds on, including the
  hooks (`authenticate`, `storage_init`, `template_object_loginform`)
  used there.
- [SSSD AD Provider](https://sssd.io/docs/ad/ad-provider.html): the
  authoritative source for `access_provider`, `simple_allow_groups`,
  and every other `sssd.conf` setting from Step 3.
- [MIT Kerberos: Developing with GSSAPI](https://web.mit.edu/kerberos/krb5-latest/doc/appdev/gssapi.html):
  covers the S4U2Self/S4U2Proxy protocol transition and constrained
  delegation extensions that Step 9's whole design rests on.
