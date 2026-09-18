# Phase 0 - Container Fundamentals

I studied containerization well enough to deploy a credential store safely, then
locked the architecture decisions the later phases depend on.

**Status:** complete.

---

## Scope

I scoped the study against what this deployment actually does. It runs a
prebuilt third-party image rather than building one, so the parts that matter
operationally are volumes, port publishing, networks, Compose, image provenance,
tag pinning, and how the daemon relates to the host.

I kept the build-side material. I need it to read the upstream Dockerfile, and
it transfers to cloud work. It is not what makes this deployment safe.

---

## Topics covered

**Core model:** containerization against virtualization, namespaces and cgroups,
images against containers, layers, the writable container layer, registries and
repositories, tags as mutable pointers, the client and daemon and socket
architecture.

**Build side:** Dockerfile instructions, layer caching and instruction ordering,
multi-stage builds, build-context exclusion.

**Runtime side:** named, anonymous, and bind-mount volumes. Port publishing and
host-to-container mapping. Container networking and name resolution. Compose
services, volumes, and networks.

**Security:** official and minimal base images, version and digest pinning,
least-privileged container users, image vulnerability scanning, image provenance
and supply-chain risk.

**Labs:** TryHackMe *Intro to Containerization* and *Intro to Docker*, plus
sandbox practice on a disposable host.

---

## What the standard material left out

General Docker courses are written for developers building their own images.
Four things they skipped turned out to matter here. I added each to my notes and
mapped it to the phase it affects.

**Interface binding on published ports.** A published port binds all interfaces
by default. I will bind the application port to one interface. That is what
scopes reachability. The container-side port only decides where traffic lands
inside the container, not who can send it.

**The container daemon writes its own packet-filter rules.** They are evaluated
ahead of the host firewall, so publishing a port can bypass a firewall that
reports default deny. I added a mandatory re-test of the Phase 4 isolation
criteria after installing the runtime instead of assuming the Phase 3 rules
still hold.

**Internal networks.** A container network can be declared with no outbound
route. It is the container-scale version of a private subnet.

**Daemon group membership is root-equivalent.** A member can mount any host path
into a container and read anything on the system. I will decide whether to grant
it in Phase 5 rather than adding my user by default.

---

## Named volume over bind mount

I evaluated both.

Named volumes inherit ownership and permissions from the image, which avoids a
common write-permission failure at first run. They are also scope-constrained to
the daemon's managed storage. A bind mount can reference any host path,
including sensitive system directories.

Bind mounts put the host path under my control and keep it visible, which
simplifies backup. They also survive prune and teardown commands that delete
named volumes.

I chose the named volume. Permission handling is a concrete, recurring benefit.
I accepted the accidental-deletion risk and mitigated it with tested backups,
which Phase 9 requires anyway. Backups are the correct control for that risk,
not mount type.

Phase 9 records the volume's real host path, because a backup target I cannot
point at is a backup target I cannot back up.

---

## SQLite over a client/server database

Early research suggested I would need a client/server database once family users
were added. That conflated two different numbers.

Family members are clients of the application, not of the database. The
application is a single process and the only writer to the database file. Six
users means one database writer. Write volume for a password manager runs to a
few dozen operations a day at family scale, against an engine that handles
thousands a second on this hardware. Vault contents are encrypted client-side,
so the server stores small ciphertext blobs and does sync bookkeeping.

Write-ahead logging is on by default and is what makes concurrent read and write
access safe. The documentation that recommends disabling it applies to network
filesystems, where file locking is unreliable. My storage is local, so I left it
on.

A second database service would add a container, a credential set, a patch
target, and a backup with consistency requirements. The decision is also
reversible, since migration tooling exists in both directions. Single-file
storage makes restore testing cheap enough that it will actually get done.

One item carried into Phase 9. With write-ahead logging active, a plain file
copy of a live database can capture an inconsistent snapshot. Backups use the
engine's consistent-snapshot mechanism or stop the container first.

---

## Mesh VPN and HTTPS together

I first treated these as alternatives. They are not.

The VPN encrypts transport. It does not give the application a TLS certificate,
and the clients require one. The web vault needs a secure context for the Web
Crypto API, which performs the actual vault encryption and decryption. The
mobile client refuses non-HTTPS server URLs. WebAuthn and FIDO2 need a secure
context.

The VPN provider issues publicly trusted certificates for tailnet hostnames, so
I get HTTPS with no public exposure and no custom CA.

That put reverse proxies, certificate automation, and web application firewalls
out of scope. It removes a large share of the upstream documentation as a direct
result of the VPN decision.

One dependency recorded. HTTPS has to work before Phase 8, because the mobile
client will reject the server otherwise.

---

## Segmentation rather than an air gap

I originally specified an air gap. An air-gapped host has no network
connectivity, which removes the reason to run a syncing password manager at all.

I restated the requirement. The host should be reachable by a chosen set of
devices, over one path, and by nothing else. That is network segmentation plus
least-privilege ingress. A host firewall bound to the VPN interface and a VPN
access-control policy achieve it at no hardware cost.

### VLAN segmentation deferred

VLAN isolation needs two things, not one. A managed switch that supports 802.1Q
tagging, and a router that can route and filter between VLANs. The host still
needs outbound access for updates and VPN coordination, so the segment cannot be
a dead end. A switch on its own accomplishes nothing, and a managed switch
behind a VLAN-unaware consumer router produces isolated segments with no
internet access. The real cost is router replacement.

I deferred it to a separate project. The public-facing part of this home lab is
the better candidate, because it accepts unsolicited inbound traffic and needs a
DMZ-style segment. A VPN-only credential store does not.

---

## Best practices

These bind the rest of the build.

- Pin the application image to a specific version tag, never a floating tag
- Verify image provenance before pulling. The upstream image is
  community-maintained, which makes identifying the canonical source the
  highest-value supply-chain check in this build
- Run as a least-privileged user and verify the UID the container actually uses
- Scan the image before deployment
- Configure exclusion files before the first commit. No secrets in the
  repository
- Never use the volume-removing teardown flag on this stack

---

## Outcome

The conclusion I carried into the rest of the build is procedural. Verify
controls by testing them, not by reading their configuration. That is why Phase
4 exists on its own and why Phase 5 runs it again.

---

[README](../README.md) · [Phase 1](phase-01-host-provisioning.md)
