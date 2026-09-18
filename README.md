# Self-Hosted Password Manager on Dedicated Hardware

Vaultwarden in a container on a dedicated single-board computer. Remote access
runs over a private mesh VPN. Built for daily use on phone, desktop, and family
accounts.

**Status:** Phase 2 of 12, host hardening.

---

## Architecture

| Area | Decision | Reason |
|---|---|---|
| Hardware | Dedicated single-board computer | Smallest attack surface and blast radius |
| Application | Vaultwarden | Free organization sharing. Cryptography stays in the official clients |
| Remote access | Mesh VPN | No public exposure, no port forwarding, no reverse proxy |
| Ingress control | Host firewall bound to the VPN interface | The local network cannot reach the service |
| Database | SQLite with write-ahead logging | Single-writer access pattern. Single-file backup and restore |
| Storage | Named container volume | Permissions inherited from the image. Scope-constrained |
| Transport | HTTPS over the tailnet | Required by the clients. Complements the VPN rather than replacing it |
| OS | Vendor Debian-based distribution, Lite, 64-bit, headless | Vendor-maintained kernel and firmware. Minimal package surface |

---

## Build plan

The host is fully hardened and verified before I store a single real credential
on it. I confirm every control by attempting the connection it should block,
not by reading its configuration.

### Phase 0 - Container fundamentals (complete)

Containers against virtualization, images and layers, Dockerfiles, Compose,
networking, volumes, container security practices. TryHackMe rooms plus sandbox
practice. Storage, database, and transport decisions locked.
[Detail](phases/phase-00-container-fundamentals.md)

### Phase 1 - Host provisioning (complete)

Flashed Lite 64-bit headless. Key-only SSH from first boot. Non-default
username. Full patch. DHCP reservation.
[Detail](phases/phase-01-host-provisioning.md)

### Phase 2 - Host hardening (in progress)

- ed25519 key-only SSH, password authentication disabled
- Root login locked
- Host firewall enabled, default deny inbound
- Intrusion prevention watching the SSH service
- Unattended security upgrades enabled and verified
- Private key backed up to a second location

### Phase 3 - Mesh VPN and network isolation

- Install the VPN client and join the tailnet
- MFA on the VPN account and on its identity provider
- Handle key expiry so the host does not silently drop off
- Access-control policy that tags the host and restricts which devices reach it
- Interface-aware firewall rules. Inbound on the VPN interface only, nothing
  inbound on the wired interface

### Phase 4 - Isolation verification

- Attempt SSH and the application port from a LAN device and confirm refusal
- Port scan the host from the LAN and confirm nothing answers
- Confirm the host is unreachable from the public internet
- Record the results

### Phase 5 - Container runtime installation

- Install the container engine and Compose plugin
- Evaluate daemon group membership as a privilege-escalation path before
  granting it
- Publish a port from a throwaway container, then re-run the Phase 4 tests. The
  daemon writes packet-filter rules ahead of the host firewall

### Phase 6 - Application deployment

- Compose file with a named volume and the port bound to the VPN interface only
- Strong admin token
- Public signups disabled after account creation
- HTTPS for the web vault over the tailnet

### Phase 7 - Application hardening

- Strong master passphrase, KDF raised to Argon2
- 2FA on the vault account
- Admin panel restricted or disabled after setup
- Intrusion prevention jail on application logs
- Update cadence defined and documented

### Phase 8 - Client rollout

- Mobile app, self-hosted URL, system autofill provider
- Desktop app and browser extension
- Confirm sync on and off the home network
- Migrate personal credentials

### Phase 9 - Backups

- Destination and method for the application data directory
- Encrypt the backups
- Automate the schedule
- Run a real restore test to a fresh container

### Phase 10 - Family rollout

- Individual accounts with separate master passwords
- Sharing through organization collections
- Onboarding walkthrough for each user

### Phase 11 - Documentation

- Build and hardening log
- Architecture diagram
- Publish with no secrets committed

### Phase 12 - Ongoing hardening (continuous)

**Weekly:** review intrusion-prevention and application logs for failed logins
and admin panel hits. Confirm unattended upgrades ran. Verify the latest backup
exists and is non-zero.

**Monthly:** patch and reboot if a kernel update landed. Pull updated images,
redeploy, verify the service. Read release notes for security fixes before
updating. Audit the VPN device list and access-control policy against reality.

**Quarterly:** full restore test to a fresh container. Re-run the Phase 4
isolation tests. Review firewall rules and listening ports for drift. Audit user
accounts and authorized keys. Review the vault for weak, reused, or breached
credentials.

**Annually:** rotate the SSH key and admin token. Rotate the master passphrase
if warranted. Review whether the architecture still fits. Update documentation
to match the build.

**Standing rules**

- Never expose the host to the public internet, including temporarily
- Never commit secrets, tokens, or keys
- Pin image versions and read release notes before updating
- Test every change against the Phase 4 isolation criteria before closing it
- Log every change with the date and the reason

---

## Repository structure

```
README.md                 Overview, architecture, build plan
phases/
  phase-00-*.md           Container fundamentals and design decisions
  phase-01-*.md           Host provisioning and troubleshooting log
```

Phase files record decisions and the reasoning behind them rather than
step-by-step instructions.

---

## Omitted from this repository

Hostnames, IP addresses, usernames, MAC addresses, key material, tokens, and
network topology. Examples use placeholders.

