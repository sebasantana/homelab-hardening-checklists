# Homelab Hardening Checklists

Practical security checklists for small homelabs, self-hosted services, and home networks.

This repository is a working notebook: a place to document repeatable hardening steps, assumptions, validation checks, and failure modes for real systems.

It is not meant to be a universal benchmark or a compliance framework. The goal is simpler:

> Make small infrastructure safer, easier to reason about, and harder to break accidentally.

These notes are written for people who operate small systems seriously, even when those systems are not enterprise-scale.

## Current status

This is an early working notebook. The checklists are intentionally marked as drafts while they are refined through real-world use and review.

## What this is not

- Not a CIS benchmark.
- Not legal, regulatory, or compliance advice.
- Not a replacement for environment-specific threat modeling.
- Not a promise that every control applies to every homelab.

## Focus areas

- Linux and macOS host baselines
- Home Assistant and smart home security
- Proxmox and virtualization hygiene
- Docker and self-hosted services
- Network segmentation and IoT isolation
- Backup, recovery, and operational checks
- Change planning and rollback discipline

## Design principles

- Prefer simple controls that are easy to verify.
- Document assumptions before applying fixes.
- Avoid changes that can lock you out without a rollback path.
- Treat backups as a security control, not an afterthought.
- Separate public examples from private infrastructure details.
- Security work should reduce uncertainty, not just add tools.

## Repository structure

```text
checklists/   Practical hardening checklists by platform or domain
templates/    Reusable templates for inventories, risks, and change plans
examples/     Sanitized examples with fictional networks and hosts
```

## Checklists

| Area | Checklist | Status |
|---|---|---|
| Linux | [Linux baseline](checklists/linux-baseline.md) | Draft |
| macOS | [macOS baseline](checklists/macos-baseline.md) | Draft |
| Home Assistant | [Home Assistant](checklists/home-assistant.md) | Draft |
| Proxmox | [Proxmox](checklists/proxmox.md) | Draft |
| Docker | [Docker](checklists/docker.md) | Draft |
| Network | [Network segmentation](checklists/network-segmentation.md) | Draft |

## How to use

1. Pick the checklist that matches the system.
2. Read the assumptions and risks before applying changes.
3. Run the discovery commands first.
4. Create a backup or snapshot when appropriate.
5. Apply changes in small batches.
6. Validate after every change.
7. Keep notes of exceptions and intentional deviations.

## Safety note

Do not paste real public IPs, secrets, customer data, private hostnames, tokens, VPN configs, or exact internal diagrams into public issues or examples.

All examples in this repository should be sanitized.

## License

MIT, unless stated otherwise.
