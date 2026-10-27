# Linux Baseline Hardening Checklist v0.3

## Scope

Local-first checklist for reviewing the baseline security of a single Linux host in a homelab, lab, or small-office environment: a VM, a bare-metal server, a Raspberry Pi, or a Docker/Proxmox host before platform-specific review.

Covers:

- inventory and discovery;
- local users, groups, and sudo;
- SSH configuration and authorized keys;
- update policy and pending reboots;
- host firewall (nftables, ufw, firewalld);
- listening services;
- filesystem permissions and secrets on disk;
- logging and journald;
- time synchronization;
- backups and restore.

This does not replace a CIS benchmark, kernel/sysctl hardening, MAC policy review (SELinux/AppArmor), EDR, vulnerability scanning, or platform-specific checklists such as [Docker](docker.md) and [Proxmox](proxmox.md).

## Assumptions

- You have authorized local or SSH access to the host, with `sudo` where noted.
- Commands are **read-only** unless explicitly stated otherwise. This checklist does not install packages, refresh package indexes, reload services, or change firewall rules.
- Examples cover Debian/Ubuntu and RHEL/Fedora families. Service names, paths, and tools differ between them; both variants are shown where it matters.
- Command output may contain sensitive data: usernames, hostnames, private IPs, key fingerprints, paths, and log lines.
- Exceptions exist: password SSH on an isolated lab VM or a passwordless sudo rule for an automation account is not always wrong, but it must be justified, documented, and compensated.

## Safety note — do not publish raw outputs

Before sharing evidence, sanitize:

- usernames, hostnames, internal domains, public IPs, sensitive private IPs;
- SSH public keys, key comments (often `user@host`), and fingerprints;
- `sudoers` entries that reveal automation accounts or tooling;
- log lines with source IPs, usernames, or URLs;
- paths containing personal or customer names.

Sanitized example:

```text
GOOD: host=app-01 ssh: PermitRootLogin no, PasswordAuthentication no, listeners=22/tcp(LAN), 443/tcp(public via proxy)
BAD:  host=<real-hostname> ssh from 203.0.113.45 user=<real-user> key comment=<real-user>@<real-laptop>
```

## Severity model

| Severity | Meaning | Typical action |
|---|---|---|
| Critical | Probable direct compromise, root access by unintended parties, or secret exposure | Fix immediately or isolate host |
| High | Significant exposure or privilege without justification | Prioritize remediation |
| Medium | Hardening gap that increases blast radius or recovery risk | Schedule fix |
| Hygiene | Documentation, consistency, maintainability | Improve when practical |

## Quick audit — 10 minutes

Use this first to decide what needs attention. Keep outputs local and sanitize before sharing.

| Check | Severity | Read-only command | Look for |
|---|---:|---|---|
| Extra UID 0 accounts | Critical | `awk -F: '$3 == 0 {print $1}' /etc/passwd` | anything other than `root` |
| Empty passwords | Critical | `sudo awk -F: '$2 == "" {print $1}' /etc/shadow` | any output |
| Effective SSH auth settings | High | `sudo sshd -T \| grep -Ei '^(permitrootlogin\|passwordauthentication\|kbdinteractiveauthentication\|pubkeyauthentication) '` | `permitrootlogin yes`, `passwordauthentication yes` on exposed hosts |
| sudo-capable users | High | `getent group sudo wheel` | unexpected members |
| Passwordless sudo | High | `sudo grep -rhE '^[^#].*NOPASSWD' /etc/sudoers /etc/sudoers.d/` | broad `ALL` rules for humans |
| Listening services | High | `sudo ss -ltnup` | `0.0.0.0` / `[::]` binds for admin or database ports |
| Host firewall present | High | `sudo nft list ruleset \| head -50` | empty ruleset, or `policy accept` on input with no rules |
| Automatic security updates | Medium | `systemctl list-timers --all 'apt-daily*' 'dnf-automatic*' 'dnf5-automatic*'` | no timer, or timer disabled |
| Pending reboot | Medium | `ls /var/run/reboot-required 2>/dev/null; uname -r` | reboot flag present, running kernel older than installed |
| Time sync | Medium | `timedatectl show -p NTPSynchronized -p Timezone` | `NTPSynchronized=no` |
| Failed units | Hygiene | `systemctl --failed --no-pager` | failed backup, timer, or security units |

> If any Critical item appears, stop treating the host as “healthy” until it is justified, isolated, or fixed.

## 1. Inventory and discovery

Goal: know what the host is, what it runs, and who depends on it before changing anything.

- [ ] **[Hygiene]** Record distribution, version, kernel, and support status (EOL releases receive no security fixes).
- [ ] **[Hygiene]** Record role, owner, criticality, and how the host is reached (console, SSH, out-of-band).
- [ ] **[Medium]** Identify running services and enabled units.
- [ ] **[Medium]** Identify network interfaces, addresses, and routes.
- [ ] **[Medium]** Confirm a console or out-of-band path exists before touching SSH or firewall.
- [ ] **[Hygiene]** Note whether the host is a VM, container host, hypervisor, or appliance; follow the matching checklist afterwards.

Read-only commands:

```bash
cat /etc/os-release
uname -r
hostnamectl
uptime
systemd-detect-virt
ip -br addr
ip route
systemctl list-units --type=service --state=running --no-pager
systemctl list-unit-files --state=enabled --no-pager
systemctl --failed --no-pager
```

Evidence to keep locally:

```text
host=<sanitized-name> os=<distro version> kernel=<version> role=<purpose> owner=<role> console=<yes/no/how> criticality=<low|medium|high>
```

## 2. Users, groups, and sudo

Goal: every account that can log in or escalate is known, needed, and individually attributable.

- [ ] **[Critical]** `root` is the only UID 0 account.
- [ ] **[Critical]** No accounts have empty password fields.
- [ ] **[High]** Members of `sudo` (Debian/Ubuntu) or `wheel` (RHEL/Fedora) are expected and named individuals or documented automation accounts.
- [ ] **[High]** `NOPASSWD` rules are scoped to specific commands, not `ALL`, unless justified.
- [ ] **[Critical]** Every sudo rule that names a command also pins its arguments, or forbids them with `""`. A binary allowed with free arguments is root if it can run commands, write arbitrary paths, or read arbitrary files: backup tools (`restic --password-command`, `restic restore --target /`, `tar`, `rsync`), `find`, editors, pagers, and interpreters all qualify (see GTFOBins).
- [ ] **[High]** Backup agents do not get sudo on the backup tool. Prefer a dedicated user with `cap_dac_read_search` on a binary only that user can execute, or a root-owned wrapper with fixed repository, paths, and no arguments.
- [ ] **[High]** Shared admin accounts are avoided; humans use named accounts.
- [ ] **[Medium]** Service accounts have a non-login shell (`/usr/sbin/nologin` or `/bin/false`).
- [ ] **[Medium]** Former users and temporary accounts are locked or removed.
- [ ] **[Hygiene]** `sudoers` drop-ins have clear file names and comments explaining intent.

Read-only commands:

```bash
awk -F: '$3 == 0 {print $1}' /etc/passwd
getent passwd | awk -F: '$7 !~ /(nologin|false)$/ {print $1, $3, $7}'
getent group sudo wheel
sudo awk -F: '$2 == "" {print $1}' /etc/shadow
sudo grep -rhE '^[^#].*NOPASSWD' /etc/sudoers /etc/sudoers.d/
sudo visudo -c          # syntax check only; does not modify files
last -n 20 -a
sudo lastb -n 20 -a 2>/dev/null   # failed logins, if btmp is kept
```

Good example:

```text
sudo group: admin-01 (named human)
/etc/sudoers.d/50-backup: backup-svc ALL=(root) NOPASSWD: /usr/local/sbin/run-backup ""
  run-backup: root:root 0755, fixed repo and paths, takes no arguments, env_reset left on
login shells: root, admin-01; all service accounts use nologin
```

Bad example:

```text
sudo group: admin, deploy, olduser, test
/etc/sudoers.d/90-deploy: deploy ALL=(ALL) NOPASSWD: ALL
/etc/sudoers.d/50-backup: backup-svc ALL=(root) NOPASSWD: /usr/bin/restic
  any arguments allowed: `sudo restic --password-command '<cmd>'` runs <cmd> as root
```

## 3. SSH

Goal: SSH accepts only the users and authentication methods you intend, from the networks you intend.

The service is named `ssh` on Debian/Ubuntu and `sshd` on RHEL/Fedora. Always check the **effective** configuration with `sshd -T`, not just `sshd_config`: drop-ins in `/etc/ssh/sshd_config.d/` are included near the top of the main file, and for most options **the first value read wins**. Cloud images often ship a drop-in such as `50-cloud-init.conf` that sets `PasswordAuthentication yes`.

On Ubuntu 22.10 and later, SSH may be socket-activated: the listen address and port come from `ssh.socket`, not from `Port`/`ListenAddress` in `sshd_config`.

- [ ] **[Critical]** Root login is disabled (`permitrootlogin no`) or limited to keys (`prohibit-password`) with a documented reason.
- [ ] **[High]** Password and keyboard-interactive authentication are disabled on any host reachable beyond a trusted admin network.
- [ ] **[High]** SSH is reachable only from admin networks or VPN, enforced by firewall, not by obscurity of port number.
- [ ] **[High]** `authorized_keys` files contain only known, current keys.
- [ ] **[Medium]** Access is restricted with `AllowUsers` / `AllowGroups` where practical.
- [ ] **[Medium]** `Match` blocks are reviewed; they can re-enable password auth for specific users or addresses.
- [ ] **[Medium]** `X11Forwarding`, `AllowAgentForwarding`, and `AllowTcpForwarding` are disabled unless needed.
- [ ] **[Hygiene]** Host keys are not shared between cloned VMs.

Read-only commands:

```bash
systemctl status ssh sshd 2>/dev/null --no-pager | head -20
systemctl status ssh.socket 2>/dev/null --no-pager | head -10
sudo sshd -T | grep -Ei '^(port|listenaddress|permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|allowusers|allowgroups|x11forwarding|allowagentforwarding|allowtcpforwarding|maxauthtries) '
# Evaluate Match blocks for a specific connection context:
sudo sshd -T -C user=<user>,host=<client-name>,addr=192.0.2.50 | grep -Ei '^(passwordauthentication|permitrootlogin) '
ls -l /etc/ssh/sshd_config.d/ 2>/dev/null
grep -nE '^\s*(Include|Match)' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/*.conf 2>/dev/null
# Count keys per authorized_keys file without printing them:
sudo find /root /home -maxdepth 3 -name authorized_keys -exec sh -c 'printf "%s keys=%s\n" "$1" "$(grep -cvE "^\s*(#|$)" "$1")"' _ {} \;
# Host key fingerprints, to compare across cloned VMs:
for k in /etc/ssh/ssh_host_*_key.pub; do ssh-keygen -lf "$k"; done
```

`sshd -T` needs root to read host keys. If it fails with a config error, the running daemon may be using an older valid config; do not restart SSH until the error is fixed and a console path is confirmed.

Good example:

```text
permitrootlogin no
passwordauthentication no
kbdinteractiveauthentication no
allowgroups ssh-admins
listen: 22/tcp, firewall allows 192.0.2.0/24 (admin VLAN) only
```

Bad example:

```text
permitrootlogin yes
passwordauthentication yes   # set by 50-cloud-init.conf, main file says "no"
listen: 0.0.0.0:22, port-forwarded from WAN
```

## 4. Updates

Goal: security fixes arrive on a predictable schedule, and reboots needed to activate them actually happen.

Refreshing the package index (`apt update`, `dnf makecache`, or a `dnf check-update` without `--cacheonly`) changes local state and contacts mirrors. The commands below read the existing cache; note its age before trusting the result.

- [ ] **[High]** The release is still supported by the distribution.
- [ ] **[Medium]** Automatic security updates are enabled, or a manual cadence is documented and followed.
- [ ] **[Medium]** Pending reboots are applied within a documented window (kernel, libc, OpenSSL, systemd updates).
- [ ] **[Medium]** Third-party repositories are known and still needed.
- [ ] **[Medium]** Auto-update scope is intentional: security-only vs all updates; automatic reboot yes/no.
- [ ] **[Hygiene]** Update failures are visible somewhere (mail, log review, monitoring).

Read-only commands, Debian/Ubuntu (`unattended-upgrades`):

```bash
systemctl list-timers --all 'apt-daily*' --no-pager
apt-config dump | grep -E '^(APT::Periodic|Unattended-Upgrade::(Allowed-Origins|Origins-Pattern|Automatic-Reboot))'
cat /etc/apt/apt.conf.d/20auto-upgrades 2>/dev/null
ls -l /var/lib/apt/lists/ | head -5          # cache age
apt list --upgradable 2>/dev/null            # uses existing cache only
sudo tail -n 30 /var/log/unattended-upgrades/unattended-upgrades.log 2>/dev/null
ls /var/run/reboot-required 2>/dev/null && cat /var/run/reboot-required.pkgs 2>/dev/null
ls /etc/apt/sources.list.d/
```

Read-only commands, RHEL/Fedora (`dnf-automatic`; Fedora 41+ ships dnf5 with the same config path):

```bash
systemctl list-timers --all 'dnf-automatic*' 'dnf5-automatic*' --no-pager
grep -E '^\s*(upgrade_type|apply_updates|reboot)\s*=' /etc/dnf/automatic.conf 2>/dev/null
dnf check-update --cacheonly                 # exit code 100 means updates are pending
dnf needs-restarting -r 2>/dev/null          # dnf-utils / dnf5 plugin; reports whether a reboot is needed
dnf repolist --cacheonly
```

Kernel cross-check (both families):

```bash
uname -r
ls -1 /boot/vmlinuz-* 2>/dev/null
```

Good example:

```text
unattended-upgrades: security origins only, Automatic-Reboot false, reboot window Sunday 04:00, last run OK
```

Bad example:

```text
release EOL since last year, no timers, 140 packages upgradable, uptime 400 days
```

## 5. Host firewall

Goal: inbound traffic is denied by default and every allowed port has a documented audience.

Most current distributions use nftables underneath. `ufw` (Ubuntu) and `firewalld` (RHEL/Fedora) are front-ends; read the front-end for intent and `nft list ruleset` for what is actually loaded. Only one front-end should manage the host.

- [ ] **[High]** A default-deny (or default-drop) inbound policy is active.
- [ ] **[High]** Each allowed port maps to a known service and audience: local-only, LAN-only, VPN-only, or public.
- [ ] **[High]** SSH and admin ports are limited to admin networks.
- [ ] **[Medium]** IPv6 is filtered as deliberately as IPv4.
- [ ] **[Medium]** IP forwarding is disabled unless the host is a router, VPN gateway, or container/VM host.
- [ ] **[Medium]** On Docker hosts, remember that published container ports are inserted ahead of ufw/firewalld rules; review them with the [Docker checklist](docker.md).
- [ ] **[Hygiene]** Firewall rules are kept in a file or config management, not only in the live ruleset.

Read-only commands:

```bash
sudo nft list ruleset
sudo ufw status verbose 2>/dev/null
sudo firewall-cmd --state 2>/dev/null && sudo firewall-cmd --get-active-zones && sudo firewall-cmd --list-all
sudo iptables -S 2>/dev/null | head -50      # legacy or iptables-nft view
sudo ip6tables -S 2>/dev/null | head -50
sysctl net.ipv4.ip_forward net.ipv6.conf.all.forwarding
```

Good example:

```text
nftables input policy drop
allow 22/tcp from 192.0.2.0/24 (admin), 443/tcp from any (reverse proxy), established/related
ipv6: same policy
```

Bad example:

```text
ufw inactive, nft ruleset empty, ip_forward=1 with no known reason
```

## 6. Listening services

Goal: nothing listens that you cannot name, and nothing listens more widely than it needs to.

- [ ] **[Critical]** Databases, caches, and admin UIs (for example PostgreSQL, MySQL, Redis, Elasticsearch, management dashboards) are not bound to all interfaces on hosts reachable from untrusted networks.
- [ ] **[High]** Every listener maps to a known process, owner, and purpose.
- [ ] **[High]** Services that only serve local clients bind to `127.0.0.1` / `::1` or a Unix socket.
- [ ] **[Medium]** Unused services are disabled through a change process, not just stopped.
- [ ] **[Medium]** UDP listeners are reviewed too (mDNS, SNMP, NTP, DNS, rpcbind).
- [ ] **[Hygiene]** Socket-activated services (`systemctl list-sockets`) are included in the inventory.

Read-only commands:

```bash
sudo ss -ltnp
sudo ss -lunp
systemctl list-sockets --no-pager
# Map a listener to its unit:
systemctl status <pid> --no-pager 2>/dev/null | head -5
```

Good example:

```text
22/tcp sshd 0.0.0.0 (firewall: admin VLAN only)
5432/tcp postgres 127.0.0.1
443/tcp caddy 0.0.0.0 (public, documented)
```

Bad example:

```text
6379/tcp redis 0.0.0.0, no auth, no firewall rule, owner unknown
111/tcp+udp rpcbind 0.0.0.0, no NFS in use
```

## 7. Filesystem permissions and secrets

Goal: credentials and keys are readable only by the accounts that need them, and nothing writable by everyone can be executed by root.

- [ ] **[Critical]** Private keys, `.env` files, and credential files are not world-readable.
- [ ] **[Critical]** Secrets are not stored in world-readable shell history, scripts, or crontabs.
- [ ] **[High]** `/etc/shadow` and SSH host private keys have restrictive permissions (typically `640 root:shadow` or `000`/`600`, and `600 root` for host keys).
- [ ] **[High]** No world-writable files in system paths, and world-writable directories have the sticky bit.
- [ ] **[Medium]** SUID/SGID binaries are inventoried; unexpected ones outside package-managed paths are investigated.
- [ ] **[Medium]** Scripts run by root (cron, systemd units, backup jobs) are not writable by non-root users.
- [ ] **[Hygiene]** Secrets locations are documented without documenting their values.

Read-only commands:

```bash
stat -c '%a %U:%G %n' /etc/passwd /etc/shadow /etc/gshadow /etc/sudoers /etc/ssh/ssh_host_*_key
# World-readable private keys and env files (keep output local):
sudo find /root /home /etc /opt /srv -xdev -type f \( -name '*.pem' -o -name '*.key' -o -name '.env' -o -name '*.env' -o \( -name 'id_*' ! -name '*.pub' \) \) -perm -o=r 2>/dev/null
# World-writable files on the root filesystem:
sudo find / -xdev -type f -perm -0002 2>/dev/null | head -50
# World-writable directories without the sticky bit:
sudo find / -xdev -type d -perm -0002 ! -perm -1000 2>/dev/null | head -50
# SUID/SGID inventory:
sudo find / -xdev -type f \( -perm -4000 -o -perm -2000 \) 2>/dev/null
# Root-run scheduled scripts and their permissions:
sudo ls -l /etc/cron.d /etc/cron.daily /etc/cron.hourly 2>/dev/null
sudo crontab -l 2>/dev/null
# Files that may contain secrets (names only; inspect locally):
sudo grep -RIlE 'password|passwd|token|secret|api[_-]?key|bearer' /etc /opt /srv 2>/dev/null | head -50
findmnt -no TARGET,OPTIONS /tmp /dev/shm /home 2>/dev/null
```

If a secret is found readable by the wrong users: treat it as compromised, rotate it, then fix permissions. Fixing permissions alone does not undo past reads.

Good example:

```text
/srv/app/.env 600 app:app
/etc/ssh/ssh_host_ed25519_key 600 root:root
no world-writable files outside /tmp, /var/tmp
```

Bad example:

```text
/opt/backup/backup.sh 777 root:root, run by root cron, contains repository password
/home/admin/.ssh/id_ed25519 644
```

## 8. Logging and journald

Goal: enough history survives a reboot or incident to reconstruct what happened.

- [ ] **[Medium]** The journal is persistent (`/var/log/journal` exists or `Storage=persistent`), or logs are forwarded elsewhere.
- [ ] **[Medium]** Journal size limits are intentional (`SystemMaxUse`), so logs neither fill the disk nor rotate away within hours.
- [ ] **[Medium]** Authentication events (SSH, sudo) are retained and reviewed after changes or incidents.
- [ ] **[Medium]** Critical hosts forward logs off-host where practical; local logs can be erased by an attacker with root.
- [ ] **[Hygiene]** Recurring warnings are understood and noted as accepted or fixed.

Read-only commands:

```bash
systemd-analyze cat-config systemd/journald.conf | grep -vE '^\s*(#|$)'
ls -ld /var/log/journal 2>/dev/null
journalctl --disk-usage
journalctl --list-boots --no-pager | tail -5
journalctl -p warning -b -n 100 --no-pager
journalctl -u ssh -u sshd --since '-7d' --no-pager | grep -Ei 'failed|invalid|accepted' | tail -30
journalctl _COMM=sudo --since '-7d' --no-pager | tail -30
```

Good example:

```text
journal persistent, SystemMaxUse=1G, 30+ days retained, auth logs forwarded to log host
```

Bad example:

```text
journal volatile (lost on reboot), no forwarding, last boot is the only history
```

## 9. Time synchronization

Goal: clocks are accurate, so logs correlate, TLS validates, and time-based tokens work.

- [ ] **[Medium]** Exactly one time-sync daemon is active (`chronyd` or `systemd-timesyncd`).
- [ ] **[Medium]** The system clock is synchronized.
- [ ] **[Hygiene]** Time sources are documented: local NTP server, router, or public pool.
- [ ] **[Hygiene]** Timezone is intentional and consistent across hosts that share logs (UTC is simplest).

Read-only commands:

```bash
timedatectl status
timedatectl timesync-status 2>/dev/null       # systemd-timesyncd
chronyc tracking 2>/dev/null                  # chrony
chronyc sources -v 2>/dev/null
systemctl is-active chronyd chrony systemd-timesyncd 2>/dev/null
```

## 10. Backups and restore

Goal: the host can be rebuilt or its data recovered within an acceptable time, by someone following written notes.

- [ ] **[High]** Data and configuration that cannot be recreated are identified (application data, databases, `/etc`, keys, crontabs).
- [ ] **[High]** Backups run on a schedule and their last success is visible.
- [ ] **[High]** At least one copy is off-host, and the backup credentials are not usable to delete all copies.
- [ ] **[High]** A restore has been tested, at least for a representative file set or database.
- [ ] **[Medium]** Databases are backed up consistently (dump or snapshot), not by copying live data files.
- [ ] **[Medium]** Rebuild notes exist: packages, config locations, restore order.
- [ ] **[Hygiene]** Backup failures alert someone.

Read-only commands:

```bash
systemctl list-timers --all --no-pager | grep -Ei 'backup|restic|borg|rsnapshot|snapshot'
ls /etc/cron.d /etc/cron.daily 2>/dev/null | grep -Ei 'backup|restic|borg|rsnapshot'
sudo crontab -l 2>/dev/null | grep -Ei 'backup|restic|borg|rsync'
journalctl -u '<backup-unit>' --since '-7d' --no-pager | tail -20
```

See the [Backup and recovery checklist](backup-recovery.md) for 3-2-1, immutability, RPO/RTO, and restore testing.

Good restore note:

```text
host=app-01
backup=restic nightly: /etc, /srv/app, postgres dump -> off-host repository
restore order=packages -> /etc -> database -> app -> firewall check
last tested=YYYY-MM-DD (file restore + db import into scratch VM)
```

Bad restore note:

```text
backup=rsync to second disk in same host
restore=not tested
```

## Final validation

Before marking the review done:

- [ ] All Critical findings are fixed, isolated, or explicitly accepted with owner and review date.
- [ ] All High findings have a remediation plan.
- [ ] Effective SSH config (`sshd -T`) matches the intended policy.
- [ ] Every listener and every firewall allow rule has a documented audience.
- [ ] Update policy and reboot window are documented.
- [ ] A restore has been tested within the documented cadence.
- [ ] No raw command output with usernames, keys, IPs, or secrets is published.
- [ ] Evidence is sanitized and stored locally.

Minimal local report template:

```text
review_date=YYYY-MM-DD
scope=<host-sanitized>
os=<distro version>
critical=<count>
high=<count>
medium=<count>
hygiene=<count>
accepted_risks=<short sanitized list>
next_review=YYYY-MM-DD
```
