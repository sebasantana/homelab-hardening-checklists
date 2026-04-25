# Proxmox Baseline Hardening Checklist

A practical review checklist for small Proxmox homelab clusters or standalone nodes.

## Assumptions

- Proxmox is used in a private lab or small internal environment.
- Availability matters, but this is not a regulated production checklist.
- Do not apply changes that can break management access without console or rollback.

## 1. Access model

- [ ] Confirm who has Proxmox admin access.
- [ ] Remove unused users and API tokens.
- [ ] Prefer named users over shared `root` usage.
- [ ] Use strong unique passwords.
- [ ] Enable MFA for admin accounts where practical.
- [ ] Restrict management UI to trusted networks/VPN.

Validation:

```text
Datacenter → Permissions → Users
Datacenter → Permissions → API Tokens
Datacenter → Permissions → Two Factor
```

## 2. Network exposure

- [ ] Confirm the Proxmox UI is not exposed directly to the public internet.
- [ ] Restrict SSH to admin networks.
- [ ] Document bridges, VLANs, and management interfaces.
- [ ] Avoid mixing untrusted VM traffic with host management.
- [ ] Review firewall state at Datacenter, Node, and VM levels.

Validation:

```bash
ip addr
ip route
ss -ltnup
pve-firewall status
```

## 3. Updates

- [ ] Confirm repository configuration.
- [ ] Review pending updates.
- [ ] Plan kernel/reboot updates carefully.
- [ ] Avoid unattended disruptive upgrades unless explicitly desired.

Validation:

```bash
apt update
apt list --upgradable
pveversion -v
```

## 4. Storage and snapshots

- [ ] Identify storage backends and capacity thresholds.
- [ ] Confirm snapshots are not being used as long-term backups.
- [ ] Review orphaned disks and unused ISOs.
- [ ] Monitor thin pool usage if using LVM-thin/ZFS.

Validation:

```bash
pvesm status
lvs || true
zpool status || true
```

## 5. Backups

- [ ] Confirm scheduled VM/container backups.
- [ ] Confirm backup retention policy.
- [ ] Store backups outside the Proxmox node where possible.
- [ ] Test restore of at least one non-critical VM.
- [ ] Document backup exclusions.

Validation:

```text
Datacenter → Backup
Datacenter → Storage
```

## 6. VM and container hygiene

- [ ] Review privileged containers.
- [ ] Review VM autostart order and dependencies.
- [ ] Confirm guest agents where useful.
- [ ] Remove stale snapshots.
- [ ] Review exposed services inside critical VMs.

## 7. Operational notes

- [ ] Document how to access the console if SSH/UI fails.
- [ ] Document shutdown/restart order.
- [ ] Document critical services and owners.
- [ ] Keep a change log for host-level modifications.

## Output template

```text
Node:
Management IP/network:
Critical VMs:
Backup status:
Top risks:
Deferred actions:
Rollback notes:
```
