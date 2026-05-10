# Home Assistant Hardening Checklist

A practical checklist for reviewing a Home Assistant instance from a security and
reliability perspective.

## Severity model

Use these labels to prioritize findings:

- **Critical**: Public exposure, unauthorized access, or broken
  safety-critical automation is likely. Fix, isolate, or disable the risky path
  immediately.
- **High**: Strong chance of account, remote-access, or recovery failure.
  Prioritize in the next maintenance window.
- **Medium**: Hardening or reliability gap with limited blast radius. Schedule
  and track.
- **Hygiene**: Documentation, cleanup, or consistency issue. Improve when
  practical.

## Assumptions

- Home Assistant is used for a private home or homelab.
- The instance may control lights, climate, sensors, energy devices, and
  automations.
- Availability and safety matter: avoid changes that can break critical
  automations without a rollback plan.

## Quick audit — 15 minutes

Use this first to decide whether the instance is safe enough for routine
maintenance. Keep screenshots and exports local unless they are sanitized.

- [ ] **[Critical] Public reachability**
  - Look at: router/firewall, reverse proxy, Nabu Casa, VPN, or tunnel config.
  - Red flags: direct port forward to HA, unknown TLS endpoint, or no
    documented owner.
- [ ] **[High] Admin users**
  - Look at: Settings → People → Users.
  - Red flags: shared admin, old temporary user, or unexpected admin.
- [ ] **[High] MFA and tokens**
  - Look at: Profile → Security.
  - Red flags: MFA disabled for admins or stale long-lived tokens.
- [ ] **[High] Backups**
  - Look at: Settings → System → Backups.
  - Red flags: no recent backup, no off-host copy, or restore never tested.
- [ ] **[Medium] Repairs/logs**
  - Look at: Settings → Repairs / Logs.
  - Red flags: auth failures, recorder errors, or broken integrations after
    restart.
- [ ] **[High] Critical automations**
  - Look at: Settings → Automations & scenes.
  - Red flags: disabled safety automation, missing entity references, or
    never-tested rollback.

Stop and treat the review as higher risk if public exposure and weak admin
authentication appear together.

## 1. Access and authentication

- [ ] **[High]** Confirm all administrator accounts are expected.
- [ ] **[High]** Remove temporary audit users after use.
- [ ] **[High]** Avoid shared admin accounts.
- [ ] **[High]** Prefer strong unique passwords.
- [ ] **[High]** Enable MFA for admin users where practical.
- [ ] **[High]** Review long-lived access tokens.
- [ ] **[High]** Revoke unused or unexplained tokens.
- [ ] **[Medium]** Confirm regular users have only the permissions they need.

Validation:

```text
Settings → People → Users
Profile → Security → Long-lived access tokens
```

## 2. Exposure and remote access

- [ ] **[Critical]** Identify how Home Assistant is reachable: LAN, VPN,
  Nabu Casa, reverse proxy, tunnel, or direct port forward.
- [ ] **[Critical]** Avoid direct public exposure unless deliberately hardened,
  monitored, and documented.
- [ ] **[High]** Prefer VPN, Tailscale, WireGuard, or Nabu Casa over raw port
  forwarding.
- [ ] **[High]** If using a reverse proxy, verify TLS, headers, trusted
  proxies, and source IP logging.
- [ ] **[High]** Confirm router/firewall rules are documented with owner,
  purpose, and review date.
- [ ] **[Medium]** Confirm mobile-app remote access behavior matches the
  intended exposure model.

Questions to answer:

```text
Is HA reachable from the public internet?
Which host terminates TLS?
Is there a documented rollback if remote access breaks?
```

## 3. Backups and recovery

- [ ] **[High]** Confirm automatic backups exist.
- [ ] **[High]** Confirm backups are recent.
- [ ] **[High]** Store at least one backup outside the HA host.
- [ ] **[High]** Test restore procedure periodically.
- [ ] **[High]** Create a backup before integration cleanup, dashboard rewrites,
  automation refactors, add-on upgrades, or database maintenance.
- [ ] **[Medium]** Record what is excluded from backups, if anything.

Validation:

```text
Settings → System → Backups
```

## 4. Repairs and logs

- [ ] **[Medium]** Review active Repairs.
- [ ] **[High]** Prioritize authentication failures, broken integrations, and
  recorder/statistics warnings.
- [ ] **[Medium]** Review recent logs after any restart or major change.
- [ ] **[Medium]** Distinguish between harmless unknown states and actual broken
  devices.
- [ ] **[Hygiene]** Keep a short note for recurring warnings that are accepted
  as non-impacting.

Validation:

```text
Settings → Repairs
Settings → System → Logs
```

## 5. Integrations and devices

- [ ] **[Medium]** List integrations with unavailable entities.
- [ ] **[Medium]** Group issues by integration before fixing individual
  entities.
- [ ] **[Medium]** Confirm whether affected devices exist physically.
- [ ] **[Medium]** Remove obsolete devices from their source platform first
  when applicable.
- [ ] **[Medium]** Avoid deleting devices from HA if the integration syncs from
  a cloud/platform source.
- [ ] **[High]** Review integrations with cloud credentials, camera access,
  locks, alarms, or climate control before broad cleanup.

Useful categories:

```text
Tuya / Smart Life
Hue
HomeKit Controller
Mobile App
Tado
MQTT
Roborock
```

## 6. Automations

- [ ] **[High]** Identify safety-critical automations: alarms, locks, climate,
  leak detection, presence-based access, and energy cutoffs.
- [ ] **[Medium]** List disabled automations and confirm if intentional.
- [ ] **[Medium]** Find automations that have never triggered.
- [ ] **[Medium]** Find stale automations not triggered in 60+ days.
- [ ] **[High]** Search logs for missing entity references in critical
  automations.
- [ ] **[High]** Protect critical automations from missing scenes/entities with
  safe defaults.
- [ ] **[Medium]** Prefer small rollback-safe edits.

Review fields:

```text
state
last_triggered
mode
referenced entities
critical actions
```

## 7. Dashboards

- [ ] **[Hygiene]** Hide unused dashboards from sidebar.
- [ ] **[Medium]** Avoid dashboards referencing obsolete entities.
- [ ] **[Hygiene]** Separate daily-use views from maintenance/debug views.
- [ ] **[Hygiene]** Prefer clear sections for operational dashboards.
- [ ] **[High]** Keep critical controls visible but not easy to trigger
  accidentally.

Suggested views:

```text
Overview
Lighting
Climate
Energy
Maintenance
```

## 8. Privacy

- [ ] **[High]** Avoid exposing location, people, cameras, internal IPs, or
  routines in public screenshots.
- [ ] **[High]** Blur names and exact locations when sharing dashboards.
- [ ] **[High]** Treat HA exports, backups, diagnostics, logs, traces, and
  `.storage` files as sensitive.
- [ ] **[Medium]** Sanitize entity names that reveal rooms, occupants, routines,
  or exact device brands before publishing examples.

## Final validation

Before marking the review done:

- [ ] All Critical findings are fixed, isolated, or explicitly accepted with
  owner and review date.
- [ ] All High findings have a remediation plan.
- [ ] A recent backup exists before any risky cleanup.
- [ ] Remote access path is documented and matches the intended exposure model.
- [ ] No raw screenshots, exports, logs, traces, or internal entity names are
  published.
- [ ] Safety-critical automations still work after changes.

## Output template

```text
Status: Good / Needs attention / Critical
Top risks:
1.
2.
3.

Actions taken:
-
-

Deferred items:
-
-

Rollback:
-

Next review:
-
```
