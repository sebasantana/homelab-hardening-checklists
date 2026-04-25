# Home Assistant Hardening Checklist

A practical checklist for reviewing a Home Assistant instance from a security and reliability perspective.

## Assumptions

- Home Assistant is used for a private home or homelab.
- The instance may control lights, climate, sensors, energy devices, and automations.
- Availability and safety matter: avoid changes that can break critical automations without a rollback plan.

## 1. Access and authentication

- [ ] Confirm all administrator accounts are expected.
- [ ] Remove temporary audit users after use.
- [ ] Avoid shared admin accounts.
- [ ] Prefer strong unique passwords.
- [ ] Enable MFA where practical.
- [ ] Review long-lived access tokens.
- [ ] Revoke unused tokens.

Validation:

```text
Settings → People → Users
Profile → Security → Long-lived access tokens
```

## 2. Exposure and remote access

- [ ] Identify how Home Assistant is reachable: LAN, VPN, Nabu Casa, reverse proxy, direct port forward.
- [ ] Avoid direct public exposure unless deliberately hardened.
- [ ] Prefer VPN, Tailscale, or Nabu Casa over raw port forwarding.
- [ ] If using reverse proxy, verify TLS, headers, and trusted proxies.
- [ ] Confirm router/firewall rules are documented.

Questions to answer:

```text
Is HA reachable from the public internet?
Which host terminates TLS?
Is there a documented rollback if remote access breaks?
```

## 3. Backups and recovery

- [ ] Confirm automatic backups exist.
- [ ] Confirm backups are recent.
- [ ] Store at least one backup outside the HA host.
- [ ] Test restore procedure periodically.
- [ ] Create a backup before integration cleanup, dashboard rewrites, or automation refactors.

Validation:

```text
Settings → System → Backups
```

## 4. Repairs and logs

- [ ] Review active Repairs.
- [ ] Prioritize authentication failures, broken integrations, and recorder/statistics warnings.
- [ ] Review recent logs after any restart or major change.
- [ ] Distinguish between harmless unknown states and actual broken devices.

Validation:

```text
Settings → Repairs
Settings → System → Logs
```

## 5. Integrations and devices

- [ ] List integrations with unavailable entities.
- [ ] Group issues by integration before fixing individual entities.
- [ ] Confirm whether affected devices exist physically.
- [ ] Remove obsolete devices from their source platform first when applicable.
- [ ] Avoid deleting devices from HA if the integration syncs from a cloud/platform source.

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

- [ ] List disabled automations and confirm if intentional.
- [ ] Find automations that have never triggered.
- [ ] Find stale automations not triggered in 60+ days.
- [ ] Search logs for missing entity references.
- [ ] Protect critical automations from missing scenes/entities.
- [ ] Prefer small rollback-safe edits.

Review fields:

```text
state
last_triggered
mode
referenced entities
critical actions
```

## 7. Dashboards

- [ ] Hide unused dashboards from sidebar.
- [ ] Avoid dashboards referencing obsolete entities.
- [ ] Separate daily-use views from maintenance/debug views.
- [ ] Prefer clear sections for operational dashboards.
- [ ] Keep critical controls visible but not easy to trigger accidentally.

Suggested views:

```text
Overview
Lighting
Climate
Energy
Maintenance
```

## 8. Privacy

- [ ] Avoid exposing location, people, cameras, internal IPs, or routines in public screenshots.
- [ ] Blur names and exact locations when sharing dashboards.
- [ ] Treat HA exports as sensitive.

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
```
