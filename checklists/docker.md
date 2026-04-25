# Docker / Self-hosted Services Checklist

## 1. Inventory

- [ ] List running containers.
- [ ] Identify exposed ports.
- [ ] Identify bind mounts.
- [ ] Identify privileged containers.
- [ ] Identify containers using host networking.

Validation:

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}'
docker inspect <container>
```

## 2. Secrets

- [ ] Avoid secrets in compose files committed to Git.
- [ ] Use `.env` files excluded by `.gitignore`.
- [ ] Rotate exposed tokens immediately.
- [ ] Avoid mounting broad host directories into containers.

## 3. Images and updates

- [ ] Prefer pinned image tags over `latest` for important services.
- [ ] Track image update cadence.
- [ ] Review changelogs before major upgrades.
- [ ] Keep rollback image/tag available.

## 4. Network exposure

- [ ] Expose only required ports.
- [ ] Put public-facing services behind a reverse proxy or VPN.
- [ ] Avoid exposing admin panels directly.
- [ ] Document which services are LAN-only, VPN-only, or public.

## 5. Persistence and backups

- [ ] Identify persistent volumes.
- [ ] Back up application data, not just compose files.
- [ ] Test restore of at least one service.
- [ ] Document restore order and dependencies.

## 6. Dangerous flags

Review containers using:

```text
--privileged
--net=host
/var/run/docker.sock mount
broad host filesystem mounts
root user inside container
```

These are not always wrong, but they should be intentional.
