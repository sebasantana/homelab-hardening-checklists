# Changelog

All notable changes to this repository are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Checklist versions in titles follow the release number (for example, release 0.3.0 ships checklists titled `v0.3`).

## [0.3.0] - 2026-10-27

### Added

- Backup and recovery checklist: 3-2-1 and off-host copies, encryption and key custody, immutability and ransomware resilience, RPO/RTO in plain language, restore order, restore testing cadence, and monitoring.
- `CHANGELOG.md`, `CONTRIBUTING.md`, and `SECURITY.md` (private reporting for accidentally published secrets or infrastructure details).
- Issue template for checklist improvements and a pull request template with a sanitization checkbox.
- Docs lint workflow (markdownlint and offline relative-link checking with lychee) and `.markdownlint.jsonc`.
- README: Backup & recovery row in the checklist table and a Versioning section.

### Changed

- Linux baseline rewritten to the full checklist format (scope, assumptions, severity model, quick audit, Good/Bad examples, final validation): inventory, users and sudo, SSH effective config (`ssh` vs `sshd` service names, drop-in precedence, socket activation), updates (`unattended-upgrades` / `dnf-automatic`), nftables/ufw/firewalld, listening services, filesystem permissions and secrets, journald, time sync, and backups.
- macOS baseline rewritten to the full checklist format: FileVault, SIP, Gatekeeper, application firewall and stealth mode, sharing services, admin users and login settings, software updates, TCC privacy permissions, login items and launch agents/daemons, Time Machine, and VM isolation notes.
- Proxmox checklist upgraded to the full checklist format, with a new Proxmox firewall section, PBS and `vzdump` backup guidance, restore testing, and LXC isolation checks.
- Docker checklist: `docker inspect $(docker ps -q)` commands in code blocks are guarded for hosts with no running containers.
- All checklist titles now carry `v0.3`.

### Fixed

- Proxmox updates validation no longer runs `apt update`; it uses read-only commands against the existing package cache and states that refreshing the index is a change.

### Removed

- `topics.txt` (repository topics are managed on GitHub).

## [0.2.0] - 2026-05-24

### Added

- Initial checklists for Linux, macOS, Home Assistant, Proxmox, Docker, and network segmentation, plus asset inventory, risk register, and change plan templates and a sanitized example (2026-04-25).
- Repository scope and draft-status notes in the README (2026-04-25).

### Changed

- Docker checklist expanded with severity model, quick audit, and sanitized examples, and language normalized to English (2026-04-26).
- Home Assistant checklist improved with severity model, quick audit, and final validation (#1, 2026-05-10).
- Network segmentation checklist improved with zones, allow-matrix, and validation evidence format (#2, 2026-05-17).
- Change plan template improved (#3, 2026-05-24).
