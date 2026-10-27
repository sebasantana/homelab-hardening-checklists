# Contributing

Thanks for helping improve these checklists. Small, well-reasoned changes are the most useful.

## Scope

- Practical hardening and recovery checks for homelabs, self-hosted services, and small networks.
- Changes that make a check more accurate, safer to run, or easier to verify.
- Not in scope: compliance mappings, vendor marketing, or controls that only make sense at enterprise scale.

## Style

Match the structure of the existing checklists (for example [Docker](checklists/docker.md) and [Network segmentation](checklists/network-segmentation.md)):

- title with version, Scope (covers / does not replace), Assumptions, Safety note, Severity model;
- a Quick audit table, then numbered sections, each with a `Goal:` line;
- items written as `- [ ] **[Critical|High|Medium|Hygiene]** ...`;
- Good and Bad example blocks;
- Final validation and a minimal local report template.

Write in English, keep items short and specific, and explain *why* when a check is not obvious.

## Commands

- Commands are **read-only by default**. Anything that changes state (refreshing package indexes, restarting services, restoring backups, toggling settings) must be labeled as a change.
- Prefer commands that show effective configuration (for example `sshd -T`) over reading a single config file.
- Note distribution or OS differences where they matter.

## Sanitize everything

- Use only documentation IP ranges: `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`.
- Use placeholders such as `<host>`, `<user>`, `<repo>`, and generic names like `app-01`.
- Never include real hostnames, domains, usernames, serial numbers, keys, tokens, screenshots, or command output from your own systems.

If you published something real by mistake, follow [SECURITY.md](SECURITY.md) instead of opening a public issue.

## Pull request flow

1. Open an issue first for larger changes (new checklist, restructured section).
2. Create a branch and keep each PR focused on one checklist or topic.
3. Run the docs lint locally if you can: `npx -y markdownlint-cli2 "**/*.md"`.
4. Fill in the pull request template, including the sanitization checkbox.
5. Add a line under an `Unreleased` heading in [CHANGELOG.md](CHANGELOG.md).
