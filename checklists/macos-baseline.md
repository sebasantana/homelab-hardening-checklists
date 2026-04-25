# macOS Baseline Hardening Checklist

## Discovery

```bash
sw_vers
uname -a
id
lsof -nP -iTCP -sTCP:LISTEN
```

## Core checks

- [ ] FileVault enabled.
- [ ] OS updates reviewed/enabled.
- [ ] Firewall state is intentional.
- [ ] Remote Login / Screen Sharing / Remote Management are intentional.
- [ ] Local admin users are expected.
- [ ] Backups are configured.
- [ ] Developer tools and automation permissions are reviewed.

## Remote access

```bash
sudo systemsetup -getremotelogin
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getstealthmode
```

## Notes

For virtual machines, document host-side isolation separately:

- NAT vs bridged networking.
- Shared folders.
- Clipboard / drag and drop.
- Port forwarding.
