<!-- markdownlint-disable-next-line MD041 -->
## Summary

What does this change and why?

## Checklists affected

- [ ] Linux baseline
- [ ] macOS baseline
- [ ] Home Assistant
- [ ] Proxmox
- [ ] Docker
- [ ] Network segmentation
- [ ] Backup and recovery
- [ ] Templates, examples, or repository docs

## Checks

- [ ] Structure matches the existing checklists (Goal lines, severity tags, Good/Bad examples where useful).
- [ ] New commands are read-only, or are clearly labeled as changes.
- [ ] Commands were verified on the stated platform/version, or the PR says they were not.
- [ ] `CHANGELOG.md` is updated under `Unreleased`.
- [ ] `npx -y markdownlint-cli2 "**/*.md"` passes locally, or CI is relied on.

## Sanitization

- [ ] I confirm this PR contains no real IPs, hostnames, domains, usernames, serial numbers, secrets, screenshots, or raw command output. Only documentation ranges (`192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`) and placeholders are used.
