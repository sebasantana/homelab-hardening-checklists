# Proxmox VE Hardening Checklist v0.3

A practical review checklist for small Proxmox VE homelab clusters or standalone nodes, including backups to Proxmox Backup Server (PBS) or plain `vzdump` storage.

## Scope

Local-first checklist for reviewing the operational security of a Proxmox VE node or small cluster in a homelab, lab, or small-office environment.

Covers:

- inventory of nodes, guests, storage, and networks;
- access model: users, realms, API tokens, MFA, permissions;
- management-plane exposure and the Proxmox firewall;
- repositories, updates, and reboots;
- storage health and snapshots;
- backups with `vzdump` and PBS, and restore testing;
- VM and container isolation hygiene;
- operational notes for recovery.

This does not replace guest-level hardening (see [Linux baseline](linux-baseline.md)), [network segmentation](network-segmentation.md) review, hardware/firmware review, or Proxmox's own documentation for clustering and Ceph.

## Assumptions

- Proxmox VE is used in a private lab or small internal environment, on a currently supported major release.
- You have authorized shell access to the node (`root` or a user with equivalent privileges) and GUI access.
- Commands are **read-only** unless explicitly stated otherwise. This checklist does not refresh package indexes, upgrade packages, change firewall state, or restore guests.
- Availability matters, but this is not a regulated production checklist.
- Do not apply changes that can break management access without console or rollback.
- Exceptions exist: a privileged LXC, PCI passthrough, or a PBS VM on the same node is not always wrong, but it must be justified, documented, and compensated.

## Safety note — do not publish raw outputs

Before sharing evidence, sanitize:

- node names, cluster names, guest names, and storage IDs that reveal context;
- management IPs, bridge and VLAN layouts, public IPs, and DNS names;
- user IDs, realm names, API token IDs, and PBS fingerprints;
- `/etc/pve` contents, `/etc/network/interfaces`, and task logs;
- PBS encryption keys and token secrets (never share these, even redacted).

Sanitized example:

```text
GOOD: pve-01 mgmt=192.0.2.10/24 (admin VLAN), UI 8006 not port-forwarded, backups -> pbs-01 (encrypted, off-node)
BAD:  <real-node>.<real-domain> UI on 203.0.113.20:8006, token root@pam!<real-token-id>, fingerprint <real-fingerprint>
```

## Severity model

| Severity | Meaning | Typical action |
|---|---|---|
| Critical | Probable full-host compromise, public management plane, or unrecoverable data | Fix immediately or isolate node |
| High | Significant exposure, broad privilege, or untested recovery | Prioritize remediation |
| Medium | Hardening gap that increases blast radius or recovery risk | Schedule fix |
| Hygiene | Documentation, consistency, maintainability | Improve when practical |

## Quick audit — 15 minutes

Use this first to decide what needs attention. Run on each node; keep outputs local and sanitize before sharing.

| Check | Severity | Read-only command | Look for |
|---|---:|---|---|
| Version and support | High | `pveversion` | major release past end of life |
| Management listeners | Critical | `ss -ltnp \| grep -E ':(22\|8006\|3128) '` | UI/SSH bound to interfaces reachable from WAN, Guest, or IoT |
| Proxmox firewall state | High | `pve-firewall status` | `disabled` with no upstream firewall protecting management |
| Users and tokens | High | `pveum user list` and `pveum user token list <userid>` | unused users, tokens without expiry, tokens with full privileges |
| Permissions | High | `pveum acl list` | `Administrator` on `/` for non-admin users or tokens |
| Repositories | Medium | `ls /etc/apt/sources.list.d/` | enterprise repo without subscription, `pvetest` on a node you depend on |
| Pending updates (cached) | Medium | `apt list --upgradable 2>/dev/null` | many pending updates, stale cache |
| Storage health | High | `pvesm status; zpool status -x 2>/dev/null` | inactive storage, near-full pools, degraded ZFS |
| Guests without backup | High | `pvesh get /cluster/backup-info/not-backed-up` | important guests listed |
| Privileged containers | High | `grep -L '^unprivileged: 1' /etc/pve/nodes/*/lxc/*.conf 2>/dev/null` | containers that are privileged without reason |

> If any Critical item appears, stop treating the node as “healthy” until it is justified, isolated, or fixed.

## 1. Inventory

Goal: know what runs where, and what depends on what, before changing anything.

- [ ] **[Hygiene]** List nodes, cluster membership, and quorum state.
- [ ] **[Hygiene]** List VMs and containers with owner, purpose, and criticality.
- [ ] **[Medium]** Identify storage backends and which guests use them.
- [ ] **[Medium]** Identify bridges, VLANs, and the management interface.
- [ ] **[Medium]** Identify guests with PCI/USB passthrough, which weakens isolation and complicates migration and restore.

Read-only commands:

```bash
pveversion
pvecm status 2>/dev/null           # cluster and quorum; errors on standalone nodes
qm list
pct list
pvesh get /cluster/resources --type vm --output-format text
pvesm status
grep -lE '^(hostpci|usb)[0-9]+:' /etc/pve/nodes/*/qemu-server/*.conf 2>/dev/null
```

`/etc/pve/qemu-server` and `/etc/pve/lxc` point only to the local node; use `/etc/pve/nodes/*/...` to cover the whole cluster.

Evidence to keep locally:

```text
node=<sanitized> version=<pve x.y> guests=<count> critical_guests=<sanitized list> storage=<types> mgmt=<192.0.2.0/24 placeholder>
```

## 2. Access model

Goal: every admin action is attributable to a named person, and automation has only the privileges it needs.

- [ ] **[High]** Confirm who has Proxmox admin access.
- [ ] **[High]** Remove unused users and API tokens.
- [ ] **[High]** Prefer named users over shared `root@pam` usage; keep `root@pam` for break-glass and cluster operations.
- [ ] **[High]** Use strong unique passwords.
- [ ] **[High]** Enable MFA (TOTP or WebAuthn) for admin accounts where practical; keep recovery keys stored off-node.
- [ ] **[High]** API tokens use privilege separation and the narrowest role and path that work (for example a backup token, not `Administrator` on `/`).
- [ ] **[Medium]** Tokens and temporary users have an expiry date.
- [ ] **[Medium]** Realms (`pam`, `pve`, LDAP/AD, OpenID) are known and unused realms removed.

Read-only commands:

```bash
pveum user list
pveum user token list <userid>
pveum acl list
pveum role list
pveum realm list
```

Validation:

```text
Datacenter → Permissions → Users
Datacenter → Permissions → API Tokens
Datacenter → Permissions → Two Factor
Datacenter → Permissions (ACLs)
Datacenter → Permissions → Realms
```

Good example:

```text
admin-01@pve: Administrator on /, TOTP enabled
backup-svc@pve!pbs: PVEDatastoreUser on /storage/pbs-01, privsep=1, expire=YYYY-MM-DD
root@pam: break-glass only, password in offline vault
```

Bad example:

```text
root@pam used daily from all devices, no MFA
automation@pam!token: privsep=0 (inherits root), no expiry, purpose unknown
```

## 3. Network exposure

Goal: the management plane (UI, API, SSH, cluster traffic) is reachable only from admin networks.

Expected listeners on a default node include `pveproxy` on 8006/tcp, `spiceproxy` on 3128/tcp, SSH on 22/tcp, `rpcbind` on 111, and local-only services such as `pvedaemon` (85/tcp on `127.0.0.1`) and Postfix (25/tcp on `127.0.0.1`). Clusters add corosync on 5405–5412/udp.

- [ ] **[Critical]** The Proxmox UI and API (8006) are not exposed directly to the public internet; remote access uses a VPN.
- [ ] **[High]** Restrict SSH and the UI to admin networks, by Proxmox firewall, upstream firewall, or `pveproxy` allow lists.
- [ ] **[High]** Avoid mixing untrusted VM traffic with host management; the management IP sits on its own VLAN or bridge.
- [ ] **[High]** Corosync and migration traffic are on trusted, preferably dedicated, networks.
- [ ] **[Medium]** Document bridges, VLANs, and management interfaces.
- [ ] **[Medium]** `rpcbind` (111) is not reachable from untrusted networks if NFS is not used.
- [ ] **[Hygiene]** Network config in `/etc/network/interfaces` matches the documentation.

Read-only commands:

```bash
ip -br addr
ip route
ss -ltnup
cat /etc/network/interfaces          # sanitize before sharing
cat /etc/default/pveproxy 2>/dev/null   # ALLOW_FROM / DENY_FROM / LISTEN_IP, if set
```

Good example:

```text
mgmt: vmbr0 192.0.2.10/24 (admin VLAN only)
guests: vmbr1 VLAN-aware, no host IP
8006/22 reachable from 192.0.2.0/24 and VPN 198.51.100.0/24 only
```

Bad example:

```text
single flat bridge for host and all guests, 8006 port-forwarded from router, IoT VLAN can reach node IP
```

## 4. Proxmox firewall

Goal: firewall state is known at every level, and enabling it cannot lock you out.

The Proxmox firewall has three levels: Datacenter (cluster-wide enable switch and rules), Node, and per-guest (VM/CT, enabled per NIC with `firewall=1`). Datacenter firewall is disabled by default. When enabled, the default input policy is DROP, but management access is still allowed from the `localnet` (the cluster network); confirm what `localnet` resolves to before relying on it.

- [ ] **[High]** Firewall state at Datacenter, Node, and VM levels is deliberate and documented.
- [ ] **[High]** `localnet` matches the admin network, not a flat LAN including untrusted devices.
- [ ] **[Medium]** Guests that face untrusted networks have per-guest firewall rules or are filtered upstream.
- [ ] **[Medium]** Security groups and IP sets are named clearly and reviewed.
- [ ] **[Hygiene]** Firewall changes follow a change plan with console access available.

Read-only commands:

```bash
pve-firewall status
pve-firewall localnet
cat /etc/pve/firewall/cluster.fw 2>/dev/null
cat /etc/pve/nodes/*/host.fw 2>/dev/null
ls /etc/pve/firewall/                # per-guest <vmid>.fw files
grep -H 'firewall=1' /etc/pve/nodes/*/qemu-server/*.conf /etc/pve/nodes/*/lxc/*.conf 2>/dev/null
```

Validation:

```text
Datacenter → Firewall → Options
Node → Firewall
VM/CT → Firewall
```

## 5. Repositories and updates

Goal: the node receives security updates from the right repository, and reboots are planned.

Refreshing the package index is a **change**: `apt update` (or Node → Updates → Refresh in the GUI) rewrites `/var/lib/apt/lists/` and contacts mirrors. Proxmox VE also runs `pve-daily-update.timer`, which refreshes the index once a day and can notify about pending updates. The commands below only read the existing cache; check its age first.

- [ ] **[High]** The major release is still supported.
- [ ] **[Medium]** Repository configuration is intentional: enterprise (with subscription) or `pve-no-subscription`; `pvetest` only on test nodes.
- [ ] **[Medium]** The Debian security repository is enabled.
- [ ] **[Medium]** Review pending updates and their source.
- [ ] **[Medium]** Plan kernel/reboot updates carefully; migrate or shut down guests in a known order.
- [ ] **[Medium]** Avoid unattended disruptive upgrades unless explicitly desired; use `apt full-upgrade` (or `apt dist-upgrade`), never plain `apt upgrade`, when you do upgrade.
- [ ] **[Hygiene]** Ceph repositories, if present, match the Ceph release in use.

Read-only commands:

```bash
pveversion -v
ls -l /etc/apt/sources.list.d/
cat /etc/apt/sources.list /etc/apt/sources.list.d/*.list /etc/apt/sources.list.d/*.sources 2>/dev/null
apt-cache policy | grep -E 'http|https' | sort -u
ls -l --time-style=long-iso /var/lib/apt/lists/ | head -5   # cache age
apt list --upgradable 2>/dev/null                            # existing cache only
systemctl list-timers pve-daily-update.timer --no-pager
uname -r
proxmox-boot-tool kernel list 2>/dev/null
```

Newer releases use deb822 `.sources` files instead of one-line `.list` files; check both.

Validation:

```text
Node → Updates
Node → Updates → Repositories
```

## 6. Storage and snapshots

Goal: storage failures are visible before they become data loss, and snapshots are not mistaken for backups.

- [ ] **[High]** ZFS pools and disks are healthy; scrubs run on a schedule.
- [ ] **[High]** Confirm snapshots are not being used as long-term backups; they live on the same storage and fail with it.
- [ ] **[Medium]** Identify storage backends and capacity thresholds.
- [ ] **[Medium]** Monitor thin pool usage if using LVM-thin/ZFS; a full thin pool can corrupt guests.
- [ ] **[Medium]** Review orphaned disks and unused ISOs.
- [ ] **[Medium]** Remove stale snapshots through a change process; old snapshots consume space and slow I/O.
- [ ] **[Hygiene]** SMART monitoring and alerting are enabled.

Read-only commands:

```bash
pvesm status
lvs 2>/dev/null
zpool status -x 2>/dev/null
zpool list 2>/dev/null
zfs list -t snapshot -o name,used,creation 2>/dev/null | head -30
smartctl -H /dev/<disk>
for id in $(qm list | awk 'NR>1 {print $1}'); do echo "vm $id"; qm listsnapshot "$id"; done
for id in $(pct list | awk 'NR>1 {print $1}'); do echo "ct $id"; pct listsnapshot "$id"; done
```

## 7. Backups (vzdump and PBS)

Goal: every guest that matters has a recent, off-node, restorable backup, and the backup system cannot be wiped with the same credentials as the node.

`vzdump` backs up guests only. It does not back up the Proxmox host itself: `/etc/pve`, `/etc/network/interfaces`, `/etc/hosts`, custom scripts, and PBS client keys must be saved separately.

General:

- [ ] **[High]** Confirm scheduled VM/container backups cover every important guest.
- [ ] **[High]** Confirm backup retention policy (keep-daily/weekly/monthly) fits the recovery need.
- [ ] **[High]** Store backups outside the Proxmox node where possible; a `local` directory backup on the same disks is not a backup of the node.
- [ ] **[High]** Backup failures send a notification that someone reads.
- [ ] **[Medium]** Mode is intentional: `snapshot` for most guests (with QEMU guest agent for filesystem freeze), `stop` for guests needing strict consistency.
- [ ] **[Medium]** Databases inside guests are also backed up at application level where crash-consistent images are not enough.
- [ ] **[Medium]** Document backup exclusions (disks with `backup=0`, excluded guests).
- [ ] **[Medium]** Host configuration is backed up separately.

PBS-specific:

- [ ] **[High]** PBS runs on separate hardware, or at minimum on separate disks with an off-site sync; a PBS VM on the node it protects does not survive that node failing.
- [ ] **[High]** Client-side encryption is enabled for backups leaving trusted storage, and the encryption key is stored **outside** both PVE and PBS (password manager or printed paper key). Without it, encrypted backups cannot be restored.
- [ ] **[High]** The PVE-side PBS credential has the least privilege that still works (for example a dedicated token with `DatastoreBackup` on one datastore or namespace, never `Admin`), and pruning is scheduled on PBS, not from PVE. Backup owners may still be able to remove their own snapshots depending on PBS version, so do not rely on this alone against a compromised node; the copy that must survive is the one in the next items.
- [ ] **[High]** At least one copy is out of reach of PVE credentials: a sync job **pulled** by a second PBS whose credentials PVE never holds, a removable datastore, or tape.
- [ ] **[High]** Verify jobs run on a schedule and report failures.
- [ ] **[Medium]** Prune and garbage-collection jobs are scheduled; datastore usage is monitored.
- [ ] **[Hygiene]** PBS itself is updated, firewalled, and has MFA on admin accounts like PVE.

Read-only commands, PVE side:

```bash
pvesh get /cluster/backup --output-format text
pvesh get /cluster/backup-info/not-backed-up --output-format text
cat /etc/pve/jobs.cfg 2>/dev/null
cat /etc/pve/vzdump.cron 2>/dev/null          # legacy schedules on older installs
grep -E '^(pbs|dir|nfs|cifs):' -A6 /etc/pve/storage.cfg | grep -Ev 'password|fingerprint'
pvesm list <backup-storage> --content backup | tail -20
ls -lt /var/log/vzdump/ 2>/dev/null | head -10
ls -l /etc/pve/priv/storage/ 2>/dev/null       # *.enc = PBS encryption key present; do not print contents
```

Read-only commands, PBS side:

```bash
proxmox-backup-manager versions
proxmox-backup-manager datastore list
proxmox-backup-manager verify-job list
proxmox-backup-manager prune-job list
proxmox-backup-manager sync-job list
proxmox-backup-manager user list
proxmox-backup-manager acl list
```

Validation:

```text
PVE: Datacenter → Backup
PVE: Datacenter → Storage
PVE: Datacenter → Notifications
PBS: Datastore → <name> → Content / Prune & GC / Verify Jobs
PBS: Configuration → Access Control
```

Good example:

```text
guests: all critical covered, nightly snapshot mode, keep-daily=7 keep-weekly=4 keep-monthly=6
target: pbs-01 (separate host), client-side encryption, key copy in offline vault
pbs: verify weekly, prune+GC daily, sync pull to pbs-02 off-site
pve token on pbs: DatastoreBackup on its own namespace; pbs-02 pulls with credentials pve never holds
host config: /etc/pve + /etc/network/interfaces archived weekly off-node
```

Bad example:

```text
target: local (same disk as guests), retention keep-last=1
pbs: VM on the same node, no encryption, credentials = root@pam
no verify, no notifications, host config never saved
```

## 8. Restore testing

Goal: a restore is a rehearsed procedure with a known duration, not a first attempt during an outage.

A test restore is a **change**: it creates a guest, consumes storage, and can cause IP, MAC, or hostname conflicts. Do it in a maintenance window with a free VMID and isolated networking.

- [ ] **[High]** Test restore of at least one non-critical VM and one container on a schedule (for example quarterly) and after major upgrades.
- [ ] **[High]** Restore to a new VMID with unique MAC addresses, NICs disconnected or on an isolated bridge, so the copy does not collide with production.
- [ ] **[High]** Confirm at least one restore works using only the off-node copy and the off-node encryption key.
- [ ] **[Medium]** Measure restore time and record it next to the RTO.
- [ ] **[Medium]** Test PBS single-file restore for a guest where that is the common case.
- [ ] **[Medium]** Document restore order: DNS/DHCP, identity, storage/NAS, databases, apps, reverse proxy.
- [ ] **[Hygiene]** Remove the test guest afterwards through the same change process.

Illustrative restore commands (these **change** state; run only as part of a planned test):

```bash
# VM to a new VMID with unique MAC addresses:
qmrestore <backup-volid> <new-vmid> --storage <target-storage> --unique 1
# Container to a new CTID:
pct restore <new-ctid> <backup-volid> --storage <target-storage> --unique 1
```

After restore and before boot, set the NIC to disconnected (`link_down=1`) or attach it to an isolated bridge in the GUI.

Evidence format:

```text
date=YYYY-MM-DD guest=<sanitized> source=<pbs-01|off-site> encrypted=<yes/no> restore_time=<minutes> boot_ok=<yes/no> app_ok=<yes/no> notes=<sanitized>
```

## 9. VM and container hygiene

Goal: a compromised guest stays inside its guest.

- [ ] **[High]** Review privileged containers; prefer unprivileged LXC, and run Docker in a VM rather than a privileged container.
- [ ] **[High]** Bind mounts (`mp0:` with host paths) and `lxc.mount.entry` lines expose only what the container needs.
- [ ] **[High]** Raw `lxc.*` overrides (`lxc.cgroup2.devices.allow`, `lxc.apparmor.profile: unconfined`, `lxc.cap.drop:` cleared) are justified.
- [ ] **[Medium]** Container `features:` (`nesting`, `keyctl`, `fuse`, `mount=`) are intentional.
- [ ] **[Medium]** Review VM autostart order and dependencies (`onboot`, `startup: order=`).
- [ ] **[Medium]** Confirm guest agents where useful (consistent backups, clean shutdown, IP reporting).
- [ ] **[Medium]** Remove stale snapshots.
- [ ] **[Medium]** Review exposed services inside critical VMs with the matching guest checklist.
- [ ] **[Hygiene]** Guest names, tags, and notes identify owner and purpose.

Read-only commands:

```bash
for f in /etc/pve/nodes/*/lxc/*.conf; do
  printf '%s ' "$f"; grep -q '^unprivileged: 1' "$f" && echo unprivileged || echo PRIVILEGED
done 2>/dev/null
grep -HE '^(mp[0-9]+|features|lxc\.)' /etc/pve/nodes/*/lxc/*.conf 2>/dev/null
grep -HE '^(onboot|startup|agent):' /etc/pve/nodes/*/qemu-server/*.conf 2>/dev/null
grep -L '^agent: 1' /etc/pve/nodes/*/qemu-server/*.conf 2>/dev/null   # VMs without guest agent enabled
```

Good example:

```text
ct 105 unprivileged, mp0=/tank/media,mp=/media,ro=1, features=nesting=1 (systemd needs it)
vm 110 agent=1, onboot=1, startup order=2
```

Bad example:

```text
ct 120 PRIVILEGED, lxc.apparmor.profile: unconfined, mp0=/,mp=/host, runs Docker
```

## 10. Operational notes

Goal: someone with these notes and console access can recover the node without guessing.

- [ ] **[High]** Document how to access the console if SSH/UI fails (IPMI/iKVM, physical keyboard/monitor, serial).
- [ ] **[Medium]** Document shutdown/restart order and cluster quorum implications (a two-node cluster loses quorum when one node is down; use a QDevice).
- [ ] **[Medium]** Document critical services and owners.
- [ ] **[Medium]** Keep the host configuration backup location and PBS encryption key location in the recovery notes, stored outside the cluster.
- [ ] **[Hygiene]** Keep a change log for host-level modifications.

## Final validation

Before marking the review done:

- [ ] All Critical findings are fixed, isolated, or explicitly accepted with owner and review date.
- [ ] All High findings have a remediation plan.
- [ ] The management plane is reachable only from admin networks or VPN.
- [ ] Every important guest is covered by a backup job (`not-backed-up` list reviewed).
- [ ] At least one restore from the off-node copy has been tested within the documented cadence.
- [ ] PBS encryption key and break-glass credentials are stored outside PVE and PBS.
- [ ] No raw output from `/etc/pve`, task logs, or network config is published.
- [ ] Evidence is sanitized and stored locally.

Minimal local report template:

```text
review_date=YYYY-MM-DD
scope=<node-or-cluster-sanitized>
pve_version=<x.y>
management_network=<placeholder>
critical_guests=<sanitized list>
backup_target=<pbs|nfs|local> off_node=<yes/no> encrypted=<yes/no> last_restore_test=YYYY-MM-DD
critical=<count>
high=<count>
medium=<count>
hygiene=<count>
accepted_risks=<short sanitized list>
deferred_actions=<short sanitized list>
rollback_notes=<short sanitized list>
next_review=YYYY-MM-DD
```
