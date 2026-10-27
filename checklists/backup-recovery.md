# Backup and Recovery Checklist v0.3

## Scope

Tool-agnostic checklist for reviewing whether a homelab, lab, or small-office environment can actually recover from disk failure, operator error, theft, or ransomware.

Covers:

- inventory of what needs to be recovered;
- 3-2-1 copies and off-host storage;
- encryption at rest and key custody;
- immutability and ransomware resilience;
- recovery objectives (RPO/RTO) in plain language;
- restore order and runbooks;
- restore testing cadence;
- monitoring of backup jobs.

This does not replace platform-specific backup sections in the [Linux](linux-baseline.md), [macOS](macos-baseline.md), [Proxmox](proxmox.md), [Docker](docker.md), and [Home Assistant](home-assistant.md) checklists, nor a formal business continuity plan.

## Assumptions

- You are reviewing systems and backup storage you own or are authorized to administer.
- Commands are **read-only** unless explicitly stated otherwise. Restore tests are changes and are called out as such.
- Examples mention common tools (restic, Borg, Proxmox Backup Server, Time Machine, NAS snapshots). Use the equivalent commands for your tooling.
- Backup listings, repository paths, bucket names, and job logs may contain sensitive data.
- Exceptions exist: a test VM with no backup, or a single copy of easily re-downloaded media, is not always wrong, but it must be a documented decision.

## Safety note — do not publish raw outputs

Before sharing evidence, sanitize:

- repository URLs, bucket names, NAS hostnames, and share paths;
- repository passwords, encryption keys, key files, and cloud credentials (never share these);
- snapshot listings that reveal hostnames, usernames, or file paths;
- job logs with source IPs or internal domains.

Sanitized example:

```text
GOOD: app-01 -> repo-a (off-host, encrypted, append-only), last snapshot YYYY-MM-DD, last restore test YYYY-MM-DD
BAD:  <real-host> -> s3:https://<real-bucket>.<real-provider>/<real-path>, RESTIC_PASSWORD=<redacted>
```

## Severity model

| Severity | Meaning | Typical action |
|---|---|---|
| Critical | Data that matters has no usable backup, or all copies can be destroyed in one event | Fix immediately |
| High | Recovery depends on untested steps, a single location, or credentials stored only on the protected system | Prioritize remediation |
| Medium | Gaps in retention, monitoring, or documentation that slow recovery | Schedule fix |
| Hygiene | Naming, consistency, review dates | Improve when practical |

## Quick audit — 15 minutes

Use this first to decide whether backups are trustworthy enough to rely on. Several checks are questions to answer from your notes; the commands are examples for common tools.

| Check | Severity | Read-only command | Look for |
|---|---:|---|---|
| Inventory exists | Critical | review notes | important data with no listed backup |
| Recent successful backup | High | `restic snapshots --latest 1` / `borg list --last 1 <repo>` / `tmutil latestbackup` | last success older than the RPO |
| Off-host copy | Critical | review job targets | every copy on the same host, disk, or site |
| Copy resistant to deletion | High | review repository and storage permissions | backup client can delete or overwrite all copies |
| Encryption | High | `restic cat config` / `borg info <repo>` | unencrypted repositories on external or cloud storage |
| Keys stored elsewhere | Critical | review password manager / offline vault | repository password only on the backed-up host |
| Restore tested | High | review restore log | no restore test, or older than the cadence |
| Job alerting | Medium | `systemctl list-timers --all \| grep -Ei 'backup\|restic\|borg'` | silent failures, nobody notified |

> If any Critical item appears, stop treating the environment as “backed up” until it is fixed or explicitly accepted.

## 1. Inventory: what must be recoverable

Goal: list what you would actually need after losing a host, not what is easy to back up.

- [ ] **[Critical]** Every system with data that cannot be recreated is listed with its backup method.
- [ ] **[High]** Configuration needed to rebuild is included: host config, compose files, `/etc`, network/firewall config, Home Assistant config, Proxmox host files.
- [ ] **[High]** Databases are captured consistently (dumps, application-aware snapshots, or stopped-service copies), not as live data files.
- [ ] **[Medium]** Data that is deliberately **not** backed up is listed with the reason (re-downloadable media, caches, scratch VMs).
- [ ] **[Hygiene]** Each entry has an owner and review date.

Inventory format:

```text
system=<sanitized> data=<what> method=<tool/mode> target=<repo-placeholder> schedule=<cron/timer> retention=<policy> rpo=<time> rto=<time> owner=<role> last_restore_test=YYYY-MM-DD
```

## 2. 3-2-1 and off-host copies

Goal: no single failure, mistake, or site event destroys every copy.

The 3-2-1 rule: at least **3** copies of data (production plus two backups), on **2** different kinds of storage or devices, with **1** copy off-site. A common extension is 3-2-1-1-0: one copy offline or immutable, and zero errors in verification.

- [ ] **[Critical]** At least one backup copy is off the host being protected.
- [ ] **[High]** At least one copy is off-site (another location, or cloud storage).
- [ ] **[High]** Copies do not share a single failure domain: same disk, same RAID array, same NAS, same power circuit, same cloud account.
- [ ] **[Medium]** RAID, ZFS mirrors, and snapshots on the same pool are treated as availability features, not backups.
- [ ] **[Medium]** Sync tools (file sync, mirrored folders) are not counted as backups unless they keep version history beyond the sync.
- [ ] **[Hygiene]** The topology of copies is drawn or written down.

Good example:

```text
copy 1: production data on app-01
copy 2: nightly to NAS (different host, snapshots enabled)
copy 3: weekly encrypted copy to off-site storage, append-only credentials
```

Bad example:

```text
copy 1: production on pve-01
copy 2: vzdump to pve-01 local storage
"copy 3": RAID1 mirror of the same disk
```

## 3. Encryption at rest and key custody

Goal: a stolen disk or leaked bucket does not expose data, and you can still decrypt your own backups after losing the host.

- [ ] **[High]** Backups on removable media, off-site storage, or cloud storage are encrypted.
- [ ] **[Critical]** Repository passwords, encryption keys, and key files are stored **outside** the system being backed up: password manager, printed paper key in a safe place, or a second trusted person.
- [ ] **[High]** Credentials needed to reach the backup storage (cloud account, NAS account, MFA recovery codes) are also available without the protected system.
- [ ] **[Medium]** Key rotation or re-keying is understood for the tool in use (some tools cannot re-encrypt old data).
- [ ] **[Medium]** Who can decrypt backups is documented; an encrypted backup readable by everyone with shell access is only partially protected.
- [ ] **[Hygiene]** The recovery notes say where keys are kept, without containing them.

Read-only commands (examples):

```bash
restic -r <repo> cat config            # shows repository version; repositories are always encrypted
borg info <repo> | grep -i encrypt     # "Encrypted: Yes (repokey)" or "No"
tmutil destinationinfo                 # then confirm encryption in System Settings
```

Borg `repokey` mode stores the key inside the repository, protected by the passphrase; `keyfile` mode stores it on the client. In `keyfile` mode, losing the client without an exported key (`borg key export`) means losing the backups.

## 4. Immutability and ransomware resilience

Goal: an attacker or a bad script with access to the production host cannot also delete or encrypt every backup.

- [ ] **[Critical]** At least one copy cannot be deleted or overwritten with credentials stored on the production host.
- [ ] **[High]** Backup clients use append-only or write-only access (for example restic/Borg repositories behind an append-only server, PBS tokens without prune rights, object storage with object lock or versioning, NAS snapshots the client cannot delete).
- [ ] **[High]** Pruning and retention run from the backup server or a separate admin context, not from each client.
- [ ] **[High]** Backup storage admin accounts are separate from production admin accounts and protected by MFA.
- [ ] **[Medium]** An offline copy (rotated external disk, tape) exists for data you cannot lose.
- [ ] **[Medium]** Retention is long enough to reach back before a slow, unnoticed compromise or corruption (weeks, not days).
- [ ] **[Medium]** Sudden growth or change rate in backups is visible; mass encryption often shows up as an unusual backup size.
- [ ] **[Hygiene]** The immutability mechanism and its retention window are documented.

Good example:

```text
clients: append-only credentials, cannot delete
server: prune weekly from admin account with MFA
off-site: object lock 30 days
offline: external disk rotated monthly
```

Bad example:

```text
clients mount the NAS share read-write with an admin account and run prune themselves
same password for NAS admin, hypervisor, and backup repository
```

## 5. Recovery objectives (RPO and RTO)

Goal: everyone agrees how much data loss and downtime is acceptable, and backups are designed to meet it.

In plain language:

- **RPO (recovery point objective):** how much recent work you can afford to lose. Nightly backups mean an RPO of up to 24 hours.
- **RTO (recovery time objective):** how long the system can be down before it really hurts. It includes finding the notes, getting hardware, downloading backups, restoring, and checking.

- [ ] **[High]** Each critical system has an RPO and RTO written down, even roughly (“lose at most a day”, “back within a weekend”).
- [ ] **[High]** Backup schedules meet the RPO.
- [ ] **[Medium]** Measured restore times (from tests) are compared with the RTO.
- [ ] **[Medium]** Restore bandwidth from off-site storage is accounted for; downloading terabytes can take days.
- [ ] **[Hygiene]** Objectives are reviewed when a system becomes more or less important.

Example:

```text
system=home-automation rpo=24h rto=4h  (manual fallback for heating exists)
system=photo-library   rpo=24h rto=7d  (irreplaceable data, slow restore acceptable)
system=media-cache     rpo=none rto=none  (not backed up by decision)
```

## 6. Restore order and runbooks

Goal: during an outage, nobody has to work out dependencies from memory.

- [ ] **[High]** Restore order is documented, starting with what everything else needs.
- [ ] **[High]** The runbook is readable without the systems it describes (printed copy, off-site note, or password manager attachment).
- [ ] **[Medium]** Each critical system has a short restore runbook: where backups are, which credentials, which commands, how to verify.
- [ ] **[Medium]** Hardware or VM requirements for restore are noted (disk size, CPU architecture, passthrough devices).
- [ ] **[Hygiene]** Runbooks are updated after each restore test.

Typical restore order:

```text
1. Network: router/firewall config, DNS, DHCP
2. Identity and secrets: password manager access, MFA recovery, backup repository keys
3. Hypervisor / storage: Proxmox host config, NAS, PBS
4. Core services: databases, reverse proxy, VPN
5. Applications: Home Assistant, self-hosted apps
6. Clients and nice-to-have services
```

Runbook template:

```text
System:
Backup location: <placeholder>
Credentials needed (where stored, not the values):
Prerequisites:
Restore steps:
Verification:
Expected duration:
Last tested: YYYY-MM-DD
```

## 7. Restore testing

Goal: prove backups work before you need them, and learn how long recovery takes.

A restore test is a **change**: it writes data, creates guests or files, and can cause address or hostname conflicts. Restore into an isolated location (scratch directory, new VMID with disconnected network, separate container) and remove it afterwards.

- [ ] **[High]** Restore at least one representative file set or system on a fixed cadence.
- [ ] **[High]** At least one test uses only the off-site copy and keys retrieved from outside the environment.
- [ ] **[High]** Test after major changes: new backup tool, new storage, OS or hypervisor upgrade, key rotation.
- [ ] **[Medium]** Verification is more than “restore finished”: files open, database starts, application logs in.
- [ ] **[Medium]** Repository integrity checks run periodically (`restic check`, `borg check`, PBS verify jobs).
- [ ] **[Hygiene]** Results and durations are recorded.

Suggested cadence:

| What | Cadence |
|---|---|
| Single-file or folder restore | Monthly |
| Full restore of one representative system | Quarterly |
| Restore from off-site copy with off-site keys | Twice a year |
| Full restore order walkthrough (tabletop or real) | Yearly |

Read-only commands (examples):

```bash
restic -r <repo> snapshots --latest 5
restic -r <repo> ls latest <path> | head
restic -r <repo> check                  # structure check; takes a non-exclusive repository lock
borg list --last 5 <repo>
borg check --repository-only <repo>     # repository consistency; can be slow
tmutil latestbackup
```

`restic check --read-data` and `borg check --verify-data` read every pack and can take hours and transfer the full repository from remote storage; schedule them deliberately.

Evidence format:

```text
date=YYYY-MM-DD system=<sanitized> source=<local|off-site> keys_from=<vault|paper> scope=<files|db|full> duration=<minutes> verified=<how> result=<pass|fail> notes=<sanitized>
```

## 8. Monitoring and retention

Goal: a failed or silently shrinking backup is noticed within one backup cycle.

- [ ] **[High]** Backup failures notify someone through a channel that is read (mail, push, monitoring).
- [ ] **[Medium]** Absence of success is detected, not just explicit failure (a job that stops running sends no error).
- [ ] **[Medium]** Retention policy is documented and matches the RPO and ransomware window.
- [ ] **[Medium]** Backup storage capacity is monitored; full storage often means silent failures.
- [ ] **[Hygiene]** Job names and repository names are consistent and sanitized in shared docs.

Read-only commands (examples):

```bash
systemctl list-timers --all --no-pager | grep -Ei 'backup|restic|borg'
journalctl -u '<backup-unit>' --since '-7d' --no-pager | tail -30
df -h <backup-mount>
```

## Final validation

Before marking the review done:

- [ ] All Critical findings are fixed or explicitly accepted with owner and review date.
- [ ] All High findings have a remediation plan.
- [ ] Every important system has a backup method, an off-host copy, and an RPO/RTO.
- [ ] At least one copy cannot be deleted with production credentials.
- [ ] Encryption keys and storage credentials are stored outside the systems being backed up.
- [ ] A restore has been tested within the documented cadence, including one from off-site.
- [ ] Restore order and runbooks are available without the systems they describe.
- [ ] No raw repository URLs, keys, or listings are published.

Minimal local report template:

```text
review_date=YYYY-MM-DD
scope=<environment-sanitized>
systems_in_scope=<count>
systems_without_offhost_copy=<count>
immutable_or_offline_copy=<yes/no>
keys_stored_outside=<yes/no>
last_full_restore_test=YYYY-MM-DD
critical=<count>
high=<count>
medium=<count>
hygiene=<count>
accepted_risks=<short sanitized list>
next_review=YYYY-MM-DD
```
