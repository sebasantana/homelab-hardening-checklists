# Change Plan

Use this template before applying a homelab change that can affect access,
security, availability, privacy, backups, or automations. Keep public copies
sanitized: no real public IPs, secrets, tokens, customer data, exact hostnames,
VPN configs, screenshots, or private diagrams.

## Goal

What are we trying to improve?

Success criteria:

- TODO

## Scope

Systems affected:

- TODO

Out of scope:

- TODO

Change owner:

Reviewer, if any:

Planned maintenance window:

## Risk

Potential failure modes:

- TODO

Risk rating:

- Critical / High / Medium / Low

Pre-change stop conditions:

- No current backup or snapshot for systems that need one.
- No rollback path for network, identity, storage, firewall, or remote-access
  changes.
- Change depends on credentials, hostnames, diagrams, or logs that have not
  been sanitized before sharing.
- Change can affect safety-critical automations and no manual fallback exists.

## Backup / rollback

Backup created:

Backup location:

Restore test status:

Rollback steps:

1. TODO
2. TODO

Rollback owner:

## Implementation steps

1. TODO
2. TODO
3. TODO

## Validation

Expected result:

Commands or UI checks:

```text

```

Post-change checks:

- [ ] Access still works from the intended admin device or path.
- [ ] No unexpected public exposure was introduced.
- [ ] Relevant logs show no repeated auth, TLS, service, or integration errors.
- [ ] Backup/snapshot state is still healthy.
- [ ] Monitoring, alerts, or manual checks confirm the service is stable.
- [ ] Any known exception has an owner and review date.

## Communication

Who needs to know:

- TODO

Short status update:

```text
Status: Planned / Completed / Rolled back / Deferred
Impact:
Validation:
Rollback:
Open risks:
```

## Publish safety check

Before using this plan in a public issue, PR, blog post, or example:

- [ ] Replace real hostnames with fictional names such as `example-host`.
- [ ] Replace internal networks with documentation ranges such as
  `192.0.2.0/24`, `198.51.100.0/24`, or `203.0.113.0/24` when possible.
- [ ] Remove public IPs, domains, tokens, API keys, VPN material, and serial
  numbers.
- [ ] Remove screenshots or verify they are redacted.
- [ ] Confirm the plan describes patterns, not a live private environment.
