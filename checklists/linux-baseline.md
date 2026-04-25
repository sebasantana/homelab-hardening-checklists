# Linux Baseline Hardening Checklist

## Discovery first

```bash
uname -a
cat /etc/os-release
id
ss -ltnup
systemctl --failed
```

## Core checks

- [ ] Security updates are enabled or reviewed regularly.
- [ ] SSH is key-based where possible.
- [ ] Root SSH login is disabled unless explicitly required.
- [ ] Firewall policy is documented.
- [ ] Listening services are expected.
- [ ] Backups are configured and restorable.
- [ ] Critical logs are reviewed.
- [ ] Secrets are not stored in world-readable paths.

## Validation

```bash
sudo systemctl status ssh || true
sudo ss -ltnup
sudo journalctl -p warning -n 100
```
