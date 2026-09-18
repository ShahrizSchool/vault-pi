# Phase 1 - Host Provisioning

I provisioned a headless host reachable over key-only SSH at a reserved address,
fully patched. Hardening is a separate phase so it starts from a baseline I know
is good.

**Status:** complete.

---

## Distribution

I chose the hardware vendor's Debian-based distribution over general-purpose ARM
builds. It ships vendor-maintained firmware and kernel, which cuts down boot,
thermal, and peripheral surprises. It also means troubleshooting results match
the configuration I am actually running.

- **Lite**, no desktop environment. Smaller attack surface, fewer packages to
  patch
- **64-bit**, which matches available RAM and provides arm64 container images
- **Headless**, with no display attached in normal operation

I adapted the hardening steps from an existing Ubuntu Server build instead of
copying them. The concepts transfer. The commands and paths do not.

---

## Key management

I generated a dedicated keypair for this host instead of reusing an existing
one. A key reused across hosts means one compromised key compromises every host
at once. Per-host keys let me revoke access to one system without touching the
others. The same principle shows up again in the VPN access-control policy and
in the container's runtime user.

### Credential separation

| Secret | Protects |
|---|---|
| SSH key passphrase | The private key file on the workstation |
| Host user password | Privilege escalation on the host |
| Vault master passphrase | Vault contents, not server-recoverable |

I set three distinct secrets. Reusing one collapses independent controls into a
single point of failure. An offline crack of a stolen key file would otherwise
also hand over the host's privilege-escalation password.

I keep host credentials in an offline copy rather than only inside the vault
they protect. A vault running on a host whose credentials live only in that
vault is unrecoverable if the host fails.

### SSH agent

On this workstation platform the SSH agent persists unlocked keys across
reboots, in OS-protected storage tied to the user account, rather than per
session. Once a key is added, its passphrase stops working as a control and
protection shifts entirely onto workstation login security.

I do not use the agent persistently. I enter the passphrase per session. Where
friction is high I add the key at the start of a work session and clear it
explicitly at the end.

---

## Provisioning

1. Assembled the hardware and installed passive and active cooling before the
   enclosure
2. Generated the keypair with an explicitly specified output filename
3. Flashed the OS image with the vendor imaging tool
4. Applied pre-boot customisation. Hostname, locale and keyboard layout,
   non-default username, password, SSH enabled with public-key authentication
   only, public key supplied
5. Configured wired networking only and left wireless unconfigured
6. Disabled the vendor remote-access service
7. Verified the generated first-run configuration on the boot partition before
   first boot
8. Booted, connected over SSH, patched fully, rebooted
9. Reserved the host's address at the gateway's DHCP server
10. Rebooted and confirmed SSH at the reserved address

Steps 7 and 10 separate a provisioned host from a host I assume is provisioned.

### Supporting decisions

**Disabled the vendor remote-access service.** The imaging tool offers a hosted
service that tunnels through vendor infrastructure. The project already has one
chosen remote-access path. A second one adds a vendor to the threat model and a
second credential set protecting the same host.

**Did not use the default username.** The distribution's historical default is a
known value and a free guess for automated attacks. This is defence in depth
rather than a primary control, since key-only authentication is what protects
SSH here, but it costs nothing.

**Wiped the vendor-supplied storage media.** The bundled card shipped with a
pre-installed OS loader. I overwrote it with an image downloaded from the
official source. A pre-installed image is code I did not choose, cannot audit,
and that sat in retail inventory for an unknown period. It is the same
provenance principle I apply to the container image in Phase 6.

**Restricted the SSH client configuration.** I defined a per-host alias with an
explicit identity file and restricted authentication to that identity alone.
Otherwise the client offers every available key in turn until one is accepted,
which discloses my key inventory to every host I contact, including untrusted
ones.

---

## Network finding, gateway status interface

I found this while configuring the DHCP reservation.

The ISP-supplied gateway exposes its status interface to any device on the local
network without authentication. Configuration changes are gated behind a
per-unit access code printed on the device, so this is read-only access rather
than open administration. It is documented behaviour for this equipment, not a
misconfiguration.

It discloses the following to anything on the LAN, with no credential:

- Device inventory. Hostnames, IP assignments, MAC addresses, connection type
- Gateway firmware version, usable for matching known vulnerabilities
- WAN address and wireless network names

Hostnames reveal what a device is for, and MAC prefixes identify hardware
vendors, so a neutral hostname does not meaningfully hide a single-board
computer.

**Assessment.** Severity scales with how trustworthy the local network is. Where
every device is controlled and maintained, the page mostly saves an attacker
time they already had, since a foothold on the LAN could enumerate the subnet
directly. It matters more with unpatched IoT devices, guest hardware, and
machines of unknown hygiene, which is the realistic case here. The gateway is
also ISP-managed equipment subject to remote configuration and firmware updates
by the provider, so the network perimeter is not under my control.

**Mitigation.** This reinforces the Phase 3 design. A host firewall bound to the
VPN interface means the service does not answer on the local network at all, so
an attacker who learns the host exists and where it sits gains nothing from
either fact. The vault's security does not rest on a perimeter device I do not
own.

**Deferred fix.** Putting a router I control behind the gateway in passthrough
mode reduces the gateway's visible inventory to one device and moves the real
inventory onto equipment with authentication I manage. It is the same hardware
purchase as the deferred VLAN work, now with two justifications.

---

## Troubleshooting log

The first provisioning attempt failed and I rebuilt it. I recorded these because
the diagnostic reasoning is the part worth reusing.

### Key generation nearly overwrote an existing key

The generation tool prompted to overwrite an existing key at the default path,
where a key for another host already lived. Answering yes would have caused
irreversible loss of access to that host.

I now specify the output filename explicitly at generation time instead of
accepting the default and answering an overwrite prompt under pressure. A
destructive prompt involving key material gets a full stop and a directory
listing before I answer.

### Public key authentication rejected

`Permission denied (publickey)` on every attempt.

The error narrowed the problem usefully. SSH was running, the host was
reachable, and password authentication was correctly disabled. Only the key
exchange failed. A dead host returns nothing at all.

| Hypothesis | Test | Result |
|---|---|---|
| Wrong key offered | The client tries only the default filename unless told otherwise | Ruled out. I specified the key path explicitly |
| Keypair mismatch | Derived the public key from the private key and compared it to the file given to the imager | Ruled out. Identical |
| Wrong username | The imaging tool lowercases the username field | Inconclusive |
| Wrong target host | Another host exists on this network | Inconclusive |
| Key never installed | Inspect the first-run script on the boot partition | Not possible |
| Directory permissions | SSH refuses keys when `.ssh` or `authorized_keys` permissions are too permissive, with no client-side message | Untested |

The first-run configuration script deletes itself after it runs, so once the
host had booted the boot partition held no evidence of what was configured. The
verification window sits between writing the image and first boot, and I missed
it. It is now step 7 of the provisioning flow.

### No console output

No video on two displays. I confirmed the correct video port, since this
hardware outputs on one port by default, and connected the display before
power-on. Video hardware is initialised at boot from the display's
identification data, and a Lite image has no display manager to detect a later
connection.

I eventually got a console prompt. I did not isolate the root cause. The video
adapter is the likely candidate. I stopped there, because the host runs headless
and video is a convenience.

### Console login rejected

The console prompt displayed the configured hostname, which confirmed
customisation had applied. A host that ignored it would show the distribution
default. That meant the user account and key should both exist.

My leading hypothesis was keyboard layout. The console layout comes from the
locale configuration, a non-US layout remaps symbol keys, and password input
does not echo, so a password containing punctuation would type as different
characters invisibly. The diagnostic is to type the password at the username
prompt, where input is visible, and inspect the symbols.

I did not confirm it. I rebuilt instead of continuing to debug. There was no
state on the device worth preserving, and starting Phase 2 from a verified
baseline was worth more than the time a workaround would have saved.

### Host key verification failure after rebuild

SSH warned that the remote host's identity had changed and refused to connect.
Reflashing generated a new host key while my client still had the previous one
recorded.

This is the control working correctly. The client cannot tell a reflashed host
from a substituted machine at the same address, and the same warning is what
would surface a real machine-in-the-middle attack.

I removed the stale known-hosts entry and accepted the new key. That is an
explicit trust decision, not routine cleanup. The rigorous version is to read
the new fingerprint from the console and compare it before accepting. Accepting
directly, on a local network minutes after writing the image myself, is a
shortcut I took knowingly.

An entry already existed for that address from a different host, which confirmed
addresses were being reassigned on this network. That turned the DHCP
reservation from housekeeping into a requirement.

### Client configuration file not read

A configured host alias failed to resolve. The configuration file was not at the
expected path. The editor's save had not landed where I assumed, and a common
variant of this is a silently appended extension.

I created the file programmatically and verified it by reading it back. This
platform's older shell also writes a byte-order mark when using UTF-8 encoding,
which some SSH builds mishandle, so I used ASCII for a file that contains only
plain characters.

---

## Lessons carried forward

- **Verify artifacts, do not trust tools.** Confirm generated configuration
  before the verification window closes. Read files back after writing them. The
  absence of this habit caused every avoidable failure in this phase
- **Read errors for what they rule out.** `Permission denied (publickey)`
  eliminated three possibilities before I tested anything
- **Security warnings are controls.** Suppressing one is a trust decision and I
  treat it as such
- **Rebuild early.** With no valuable state on the device, a clean rebuild costs
  less than debugging a process I do not understand, and Phase 2 depends on a
  trustworthy baseline
- **Separate credentials by function** so one compromise does not cascade
- **Invisible input hides configuration errors.** Non-echoing prompts conceal
  keyboard layout problems, so test input where it is visible

---

## Exit criteria

| Criterion | Status |
|---|---|
| Key-only SSH works, password authentication never enabled | Met |
| Address reserved at the gateway and verified across a reboot | Met |
| Fully patched | Met |
| Vendor remote-access service disabled | Met |
| Private key backed up to a second location | Met |

Next is Phase 2, host hardening.

---

[Phase 0](phase-00-container-fundamentals.md) · [README](../README.md)
