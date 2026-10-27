# macOS Baseline Hardening Checklist v0.3

## Scope

Local-first checklist for reviewing the baseline security of a Mac used as an admin workstation, homelab host, or small-office machine.

Covers:

- inventory and discovery;
- disk encryption (FileVault);
- platform protections (SIP, Gatekeeper);
- application firewall and stealth mode;
- sharing services (Remote Login, Screen Sharing, Remote Management, File Sharing);
- local admin users and login settings;
- software updates;
- privacy (TCC) permissions;
- login items, launch agents, and launch daemons;
- Time Machine backups;
- VM isolation notes.

This does not replace MDM baselines, the macOS Security Compliance Project, EDR, or vendor guidance for managed fleets.

## Assumptions

- You have authorized access to the Mac with an admin account where `sudo` is noted.
- Commands are **read-only** unless explicitly stated otherwise. This checklist does not toggle settings, reset permissions, or install updates.
- Tested against recent macOS releases (Ventura and later). Some commands need Full Disk Access granted to the terminal app; when a command reports a permission error, use the System Settings path instead of granting extra access just for the checklist.
- Command output may contain sensitive data: usernames, hostnames, serial numbers, Wi-Fi names, share names, and app inventories.
- Exceptions exist: Remote Login on a headless homelab Mac, or a disabled firewall on a host behind a strict network firewall, is not always wrong, but it must be justified, documented, and compensated.

## Safety note — do not publish raw outputs

Before sharing evidence, sanitize:

- usernames, full names, computer names, and local hostnames (`*.local`);
- serial numbers, hardware UUIDs, and FileVault recovery keys (never share these);
- share names, Time Machine destination names, and network volume URLs;
- app and launch agent inventories that reveal employers, clients, or tooling;
- screenshots of System Settings that show account names or devices.

Sanitized example:

```text
GOOD: mac-01 FileVault=On SIP=enabled Gatekeeper=enabled firewall=on stealth=on remote-login=off admins=[admin-user]
BAD:  <real-name>'s MacBook serial=<real-serial> recovery key=<redacted> share=smb://<real-nas>/<real-folder>
```

## Severity model

| Severity | Meaning | Typical action |
|---|---|---|
| Critical | Probable direct compromise, unencrypted data at risk, or platform protections disabled without reason | Fix immediately or isolate machine |
| High | Significant exposure or privilege without justification | Prioritize remediation |
| Medium | Hardening gap that increases blast radius or recovery risk | Schedule fix |
| Hygiene | Documentation, consistency, maintainability | Improve when practical |

## Quick audit — 10 minutes

Use this first to decide what needs attention. Keep outputs local and sanitize before sharing.

| Check | Severity | Read-only command | Look for |
|---|---:|---|---|
| FileVault | Critical | `fdesetup status` | `FileVault is Off` on a portable or shared Mac |
| System Integrity Protection | Critical | `csrutil status` | `disabled` or custom configuration |
| Gatekeeper | High | `spctl --status` | `assessments disabled` |
| Application firewall | High | `/usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate` | `disabled` on a Mac that joins untrusted networks |
| Stealth mode | Medium | `/usr/libexec/ApplicationFirewall/socketfilterfw --getstealthmode` | `off` on portable Macs |
| Listening services | High | `sudo lsof -nP -iTCP -sTCP:LISTEN` | 22, 445, 548, 3283, 5900 when sharing should be off |
| Admin users | High | `dscl . -read /Groups/admin GroupMembership` | unexpected or shared admins |
| Automatic login | High | `defaults read /Library/Preferences/com.apple.loginwindow autoLoginUser 2>&1` | a username instead of “does not exist” |
| OS version | Medium | `sw_vers` | release no longer receiving security updates |
| Persistence items | Medium | `ls -la ~/Library/LaunchAgents /Library/LaunchAgents /Library/LaunchDaemons` | unknown vendors or leftover uninstalls |
| Time Machine | High | `tmutil destinationinfo` | no destination configured |

> If any Critical item appears, stop treating the Mac as “healthy” until it is justified, isolated, or fixed.

## 1. Inventory and discovery

Goal: know what the Mac is, who uses it, and how it is reached before changing anything.

- [ ] **[Hygiene]** Record model, architecture (Apple silicon or Intel), macOS version, and role.
- [ ] **[Hygiene]** Record owner, primary user, and whether it is managed by MDM.
- [ ] **[Medium]** Identify listening services and which sharing feature opened them.
- [ ] **[Medium]** Identify installed configuration profiles; they can enforce settings and grant privacy permissions.

Read-only commands:

```bash
sw_vers
uname -m
id
scutil --get ComputerName
lsof -nP -iTCP -sTCP:LISTEN          # current user's processes
sudo lsof -nP -iTCP -sTCP:LISTEN     # all processes
profiles status -type enrollment
sudo profiles list
```

Evidence to keep locally:

```text
host=<sanitized-name> model=<generic> arch=<arm64|x86_64> macos=<version> role=<purpose> mdm=<yes/no> owner=<role>
```

## 2. Disk encryption (FileVault)

Goal: data at rest is unreadable without user credentials or the recovery key.

- [ ] **[Critical]** FileVault is On for portable Macs and any Mac with sensitive data.
- [ ] **[High]** The recovery key is stored outside the Mac (password manager, printed copy in a safe place, or MDM escrow), never in a file on the same disk.
- [ ] **[Medium]** Only intended users are FileVault-enabled (they can unlock the disk at boot).
- [ ] **[Hygiene]** For headless Macs, the trade-off between FileVault and unattended reboot is documented.

Read-only commands:

```bash
fdesetup status
sudo fdesetup list                   # FileVault-enabled users
diskutil apfs list | grep -E 'FileVault|Volume Name|Encrypted'
```

```text
System Settings → Privacy & Security → FileVault
```

## 3. Platform protections (SIP and Gatekeeper)

Goal: the OS protects system files and blocks unsigned or unnotarized code by default.

- [ ] **[Critical]** System Integrity Protection is enabled.
- [ ] **[High]** Gatekeeper assessments are enabled.
- [ ] **[Medium]** On Apple silicon, Startup Security is “Full Security” unless a documented need (kernel extensions, older OS) requires otherwise.
- [ ] **[Medium]** Developer Tools access is intentional; it lets listed apps run unsigned code without Gatekeeper checks.

Read-only commands:

```bash
csrutil status
spctl --status
DevToolsSecurity -status
```

```text
Startup Security (Apple silicon): Recovery → Utilities → Startup Security Utility
System Settings → Privacy & Security → Developer Tools
```

Changing SIP or Startup Security requires booting into Recovery. Treat any request to disable them as a change requiring justification and a rollback plan.

## 4. Application firewall and stealth mode

Goal: inbound connections are allowed only for apps you approved, and the Mac does not answer probes on untrusted networks.

The macOS application firewall is per-application and inbound-only. It does not replace a network firewall and does not filter outbound traffic. `pf` is a separate packet filter that is usually unused unless configured.

- [ ] **[High]** The application firewall is enabled on Macs that join networks you do not control.
- [ ] **[Medium]** Stealth mode is enabled on portable Macs.
- [ ] **[Medium]** The allowed-apps list contains only apps that need inbound connections.
- [ ] **[Medium]** “Automatically allow downloaded signed software” is a deliberate choice.
- [ ] **[Hygiene]** If `pf` is used, its rules are documented.

Read-only commands:

```bash
/usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
/usr/libexec/ApplicationFirewall/socketfilterfw --getstealthmode
/usr/libexec/ApplicationFirewall/socketfilterfw --getblockall
/usr/libexec/ApplicationFirewall/socketfilterfw --getallowsigned
/usr/libexec/ApplicationFirewall/socketfilterfw --listapps
sudo pfctl -s info 2>/dev/null | head -5
```

```text
System Settings → Network → Firewall → Options
```

## 5. Sharing services

Goal: every remote access path is intentional, restricted to named users, and reachable only from trusted networks.

| Feature | Typical ports | Notes |
|---|---|---|
| Remote Login (SSH) | 22/tcp | Can be restricted to specific users; review “Allow full disk access for remote users” |
| Screen Sharing | 5900/tcp | VNC-compatible; restrict to named users |
| Remote Management (ARD) | 3283/tcp+udp, 5900/tcp | Grants broad control; superset of Screen Sharing |
| File Sharing (SMB) | 445/tcp | Review shared folders and per-user access |
| AirPlay Receiver | 5000/tcp, 7000/tcp | Often the unexplained listener on 5000/7000 |

- [ ] **[Critical]** No sharing service is reachable from the internet (port forward, UPnP, or exposed VPN-less tunnel).
- [ ] **[High]** Remote Login, Screen Sharing, and Remote Management are off unless needed, and limited to named users when on.
- [ ] **[High]** Remote Management is not enabled “for all users” with broad privileges.
- [ ] **[Medium]** File Sharing exposes only intended folders; guest access to shared folders is off.
- [ ] **[Medium]** SSH on a Mac follows the same key-only policy as Linux hosts (`sudo sshd -T` works on macOS too).
- [ ] **[Hygiene]** AirPlay Receiver, Media Sharing, Printer Sharing, and Internet Sharing are off unless used.

Read-only commands:

```bash
sudo systemsetup -getremotelogin     # may require Full Disk Access for the terminal
sudo launchctl print-disabled system | grep -E 'com.openssh.sshd|com.apple.screensharing|com.apple.smbd'
pgrep -lf ARDAgent                   # running agent suggests Remote Management is active
sharing -l                           # File Sharing share points
sudo lsof -nP -iTCP -sTCP:LISTEN | grep -E ':(22|445|548|3283|5000|5900|7000) '
sudo sshd -T 2>/dev/null | grep -Ei '^(permitrootlogin|passwordauthentication|kbdinteractiveauthentication) '
```

```text
System Settings → General → Sharing
```

In `launchctl print-disabled`, `=> enabled` (or `false`, on older releases) means the service is allowed to run. The System Settings view is the authoritative source if the two disagree.

Good example:

```text
Remote Login: on, allowed users=admin-user only, key-only auth, firewall allows 192.0.2.0/24
Screen Sharing / Remote Management / File Sharing: off
```

Bad example:

```text
Remote Management: on, all users, all privileges
File Sharing: on, guest access to Public and Downloads
5900/tcp port-forwarded from router
```

## 6. Admin users and login settings

Goal: few admins, all named, and the Mac never logs in on its own.

- [ ] **[High]** Admin group members are expected; daily use happens in a standard account where practical.
- [ ] **[High]** Automatic login is disabled.
- [ ] **[Medium]** The Guest account is disabled unless deliberately used.
- [ ] **[Medium]** Screen lock requires a password immediately or after a short delay.
- [ ] **[Medium]** Admin accounts that must unlock FileVault have a Secure Token.
- [ ] **[Hygiene]** Hidden or service accounts with UID 500+ are explained.

Read-only commands:

```bash
dscl . -read /Groups/admin GroupMembership
dscl . -list /Users UniqueID | awk '$2 >= 500'
defaults read /Library/Preferences/com.apple.loginwindow autoLoginUser 2>&1
defaults read /Library/Preferences/com.apple.loginwindow GuestEnabled 2>&1
sysadminctl -secureTokenStatus <user> 2>&1
sysadminctl -screenLock status 2>&1
```

```text
System Settings → Users & Groups
System Settings → Lock Screen
```

## 7. Software updates

Goal: OS and app security fixes are applied promptly, including Rapid Security Responses and background security data.

`softwareupdate --list` does not install anything, but it contacts Apple's update servers and refreshes the local catalog. Run it knowingly on metered or monitored networks.

- [ ] **[High]** The macOS release still receives security updates.
- [ ] **[Medium]** Automatic checks, security responses, and system data files are enabled.
- [ ] **[Medium]** Pending updates are applied within a documented window.
- [ ] **[Medium]** Third-party updaters (Homebrew, app stores, vendor updaters) are part of the review.
- [ ] **[Hygiene]** Update history shows recent installs.

Read-only commands:

```bash
sw_vers
defaults read /Library/Preferences/com.apple.SoftwareUpdate 2>/dev/null | grep -E 'AutomaticCheckEnabled|AutomaticDownload|AutomaticallyInstallMacOSUpdates|CriticalUpdateInstall|ConfigDataInstall'
softwareupdate --history | tail -15
softwareupdate --list                # network: contacts Apple, does not install
HOMEBREW_NO_AUTO_UPDATE=1 brew outdated 2>/dev/null
```

```text
System Settings → General → Software Update → Automatic Updates (ⓘ)
```

## 8. Privacy permissions (TCC)

Goal: only trusted apps hold permissions that bypass normal file and input protections.

TCC permissions are best reviewed in System Settings. Reading the TCC databases directly requires Full Disk Access and is not needed for this review. `tccutil reset` **removes** permissions; it is a change, not an audit step.

- [ ] **[High]** Full Disk Access is limited to backup, security, and admin tools you actively use.
- [ ] **[High]** Accessibility and Input Monitoring are limited to known automation or input tools; both can observe or control the session.
- [ ] **[High]** Screen & System Audio Recording is limited to apps that need it.
- [ ] **[Medium]** Automation (Apple Events) grants between apps are expected.
- [ ] **[Medium]** Camera, Microphone, and Location grants are expected.
- [ ] **[Medium]** Configuration profiles that pre-approve permissions (PPPC) are known.
- [ ] **[Hygiene]** Entries for uninstalled apps are noted for cleanup.

Review path:

```text
System Settings → Privacy & Security → Full Disk Access
System Settings → Privacy & Security → Accessibility
System Settings → Privacy & Security → Input Monitoring
System Settings → Privacy & Security → Screen & System Audio Recording
System Settings → Privacy & Security → Automation
System Settings → Privacy & Security → Files and Folders
System Settings → Privacy & Security → Camera / Microphone / Location Services
System Settings → General → Device Management (profiles)
```

## 9. Login items, launch agents, and launch daemons

Goal: everything that starts automatically is known, signed by an expected vendor, and still needed.

- [ ] **[High]** Launch daemons (run as root) in `/Library/LaunchDaemons` belong to known, installed software.
- [ ] **[Medium]** Launch agents in `/Library/LaunchAgents` and `~/Library/LaunchAgents` are known.
- [ ] **[Medium]** Login items and background items listed in System Settings are expected.
- [ ] **[Medium]** Leftovers from uninstalled apps are noted for cleanup.
- [ ] **[Hygiene]** Persistence items you add yourself (backup scripts, sync jobs) are documented.

Read-only commands:

```bash
ls -la ~/Library/LaunchAgents /Library/LaunchAgents /Library/LaunchDaemons
launchctl list | grep -v 'com.apple.'            # current user's domain
sudo launchctl list | grep -v 'com.apple.'       # system domain
# Inspect what a plist runs:
plutil -p /Library/LaunchDaemons/<label>.plist
codesign -dv --verbose=2 <path-to-program> 2>&1 | grep -E 'Authority|TeamIdentifier'
# Background Task Management inventory (macOS 13+; output is long and sensitive):
sudo sfltool dumpbtm | less
```

```text
System Settings → General → Login Items & Extensions
```

Good example:

```text
/Library/LaunchDaemons: com.vendor.backup-agent (signed, TeamIdentifier known, in use)
~/Library/LaunchAgents: local.backup-sync (documented, owner admin-user)
```

Bad example:

```text
/Library/LaunchDaemons: com.unknown.helper.plist -> /Users/Shared/.helper (unsigned, no installed app)
```

## 10. Time Machine and recovery

Goal: user data can be restored, and the backup itself does not become a data leak.

- [ ] **[High]** A Time Machine destination (or another backup tool) is configured and backups are recent.
- [ ] **[High]** Backup destinations are encrypted.
- [ ] **[Medium]** At least one copy is off the Mac and not permanently connected (rotated disk, NAS with snapshots, or cloud backup).
- [ ] **[Medium]** A restore of a sample folder has been tested.
- [ ] **[Medium]** Exclusions are intentional (VM images, caches, large media).
- [ ] **[Hygiene]** Credentials needed to restore (NAS account, encryption password, FileVault recovery key) are stored outside the Mac.

Read-only commands:

```bash
tmutil destinationinfo
tmutil latestbackup 2>&1             # may require Full Disk Access
tmutil isexcluded ~/<folder>
defaults read /Library/Preferences/com.apple.TimeMachine SkipPaths 2>/dev/null
```

```text
System Settings → General → Time Machine (encryption shown per destination)
```

See the [Backup and recovery checklist](backup-recovery.md) for 3-2-1, immutability, and restore cadence.

## 11. Virtual machines

Goal: a compromised or experimental VM cannot reach more of the host or network than intended.

For virtual machines, document host-side isolation separately:

- [ ] **[High]** NAT vs bridged networking is intentional; bridged VMs are full LAN citizens and need their own hardening.
- [ ] **[High]** Shared folders expose only the paths the VM needs, read-only where possible; never the whole home folder.
- [ ] **[Medium]** Clipboard and drag-and-drop sharing are disabled for untrusted or malware-analysis VMs.
- [ ] **[Medium]** Port forwarding from host to VM is documented and bound to `127.0.0.1` unless LAN access is intended.
- [ ] **[Medium]** VM images and snapshots containing secrets are excluded from unencrypted backups or stored encrypted.
- [ ] **[Hygiene]** Each VM has an owner, purpose, and a matching guest checklist (for example [Linux baseline](linux-baseline.md)).

Evidence format:

```text
vm=<sanitized-name> network=<nat|bridged|host-only> shared_folders=<none|path:ro> clipboard=<on|off> port_forwards=<127.0.0.1:2222->22> owner=<role>
```

## Final validation

Before marking the review done:

- [ ] All Critical findings are fixed, isolated, or explicitly accepted with owner and review date.
- [ ] All High findings have a remediation plan.
- [ ] FileVault, SIP, and Gatekeeper state is recorded.
- [ ] Every enabled sharing service has a documented user list and network audience.
- [ ] Launch daemons and privacy permissions have been reviewed.
- [ ] A recent backup exists and a restore has been tested within the documented cadence.
- [ ] No raw output with usernames, serials, recovery keys, or share names is published.
- [ ] Evidence is sanitized and stored locally.

Minimal local report template:

```text
review_date=YYYY-MM-DD
scope=<mac-sanitized>
macos=<version>
critical=<count>
high=<count>
medium=<count>
hygiene=<count>
accepted_risks=<short sanitized list>
next_review=YYYY-MM-DD
```
