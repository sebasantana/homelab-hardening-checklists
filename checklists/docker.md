# Docker / Self-hosted Services Checklist v0.2

## Scope

Checklist local-first para revisar seguridad operativa de Docker y servicios self-hosted en homelab, laboratorio o small office.

Cubre:

- inventario de contenedores, imágenes, puertos, mounts y redes;
- exposición local/LAN/VPN/pública;
- secretos y archivos sensibles;
- flags y permisos peligrosos;
- persistencia, backups y restore;
- estrategia de updates y rollback.

No reemplaza un hardening completo del host, firewall, reverse proxy, IAM, SIEM, EDR ni gestión formal de vulnerabilidades.

## Assumptions

- Tenés acceso local autorizado al host Docker.
- Los comandos son **read-only** salvo que se indique lo contrario. En esta checklist no se incluyen acciones destructivas.
- El output puede contener datos sensibles: dominios internos, rutas, nombres de servicios, variables, tokens o IPs privadas.
- Las excepciones existen: `--privileged`, `root`, `host network` o `docker.sock` no siempre son incorrectos, pero deben estar justificados, documentados y compensados.

## Safety note — no publicar outputs crudos

Antes de compartir evidencia, sanitizá:

- tokens, API keys, passwords, cookies, JWTs;
- dominios internos, hostnames, IPs públicas, IPs privadas sensibles;
- rutas locales con nombres de usuarios o clientes;
- nombres de contenedores que revelen arquitectura o negocio;
- valores de `.env`, labels de proxy, URLs internas y headers.

Ejemplo sanitizado:

```text
GOOD: app-web -> 127.0.0.1:8080, image app:1.4.2, volume /srv/app-data:/data:ro
BAD:  app-web -> 203.0.113.10:8080, TOKEN=<redacted>, /Users/alice/client-x/prod:/data
```

## Severity model

| Severity | Meaning | Typical action |
|---|---|---|
| Critical | Probable direct compromise or secret exposure | Fix immediately or isolate service |
| High | Significant exposure or dangerous privilege without justification | Prioritize remediation |
| Medium | Hardening gap that increases blast radius or recovery risk | Schedule fix |
| Hygiene | Documentation, consistency, maintainability | Improve when practical |

## Quick audit — 10 minutes

Use this first to decide what needs attention. Keep outputs local and sanitize before sharing. If there are no running containers, some `docker inspect $(docker ps -q)` commands may return an error; note it and continue. For scripts, prefer guarding empty input first, for example: `ids=$(docker ps -q); [ -n "$ids" ] && docker inspect ... $ids`.

| Check | Severity | Read-only command | Look for |
|---|---:|---|---|
| Running containers and published ports | High | `docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}'` | `0.0.0.0`, public ports, admin UIs |
| Privileged containers | Critical | `docker inspect --format '{{.Name}} privileged={{.HostConfig.Privileged}}' $(docker ps -q)` | `privileged=true` |
| Host networking | High | `docker inspect --format '{{.Name}} network={{.HostConfig.NetworkMode}}' $(docker ps -q)` | `network=host` |
| Docker socket mounts | Critical | `docker inspect --format '{{.Name}} {{range .Mounts}}{{.Source}}:{{.Destination}} {{end}}' $(docker ps -q)` | `/var/run/docker.sock` |
| Broad host mounts | High | `docker inspect --format '{{.Name}} {{range .Mounts}}{{.Source}}:{{.Destination}} {{end}}' $(docker ps -q)` | `/`, `/home`, `/Users`, `/etc`, `/var` mounted broadly |
| Secrets in compose/env | Critical | `grep -RIlE 'password|passwd|token|secret|api[_-]?key|jwt' . --exclude-dir=.git` | file names that may contain secrets; inspect locally only |
| Image tags | Medium | `docker ps --format '{{.Names}} {{.Image}}'` | important services using `latest` |
| Volumes/backups | High | `docker volume ls` + compose files | persistent data with no backup/restore note |

> If any Critical item appears, stop treating the system as “healthy” until it is justified, isolated, or fixed.

## 1. Inventory

Goal: know what is running before changing anything.

- [ ] **[Hygiene]** List running containers.
- [ ] **[High]** Identify published ports and bind addresses.
- [ ] **[High]** Identify bind mounts and named volumes.
- [ ] **[Critical]** Identify privileged containers.
- [ ] **[High]** Identify containers using host networking.
- [ ] **[Medium]** Identify restart policies and dependency order.
- [ ] **[Hygiene]** Map each container to owner, purpose and criticality.

Read-only commands:

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
docker ps --format 'table {{.Names}}\t{{.RunningFor}}\t{{.Ports}}'
docker inspect --format '{{.Name}} restart={{.HostConfig.RestartPolicy.Name}} network={{.HostConfig.NetworkMode}} privileged={{.HostConfig.Privileged}}' $(docker ps -q)
docker inspect --format '{{.Name}} {{range .Mounts}}{{.Type}} {{.Source}} -> {{.Destination}} rw={{.RW}}; {{end}}' $(docker ps -q)
```

Evidence to keep locally. Use sanitized names; do not include customer names, real public IPs, domains, tokens or full private paths:

```text
container=<sanitized-name> image=<repo:tag> ports=<bind:port> mounts=<summary> risk=<severity>
```

Good example:

```text
media-app image=vendor/app:2.8.1 ports=127.0.0.1:8096->8096/tcp mounts=/srv/media-config->/config rw=true risk=Medium
```

Bad example:

```text
admin-ui image=vendor/admin:latest ports=0.0.0.0:9000->9000/tcp mounts=/->/host rw=true privileged=true risk=Critical
```

## 2. Exposure and network

Goal: expose only what is required, to the smallest audience possible.

- [ ] **[Critical]** Admin panels are not exposed directly to the internet.
- [ ] **[High]** Published ports bind to `127.0.0.1` unless LAN/public access is intentional.
- [ ] **[High]** Public-facing services sit behind a reverse proxy, VPN, tunnel, or firewall rule with documented intent.
- [ ] **[Medium]** Services are classified as `local-only`, `LAN-only`, `VPN-only`, or `public`.
- [ ] **[Medium]** Docker networks separate unrelated stacks where practical.
- [ ] **[Hygiene]** Every public or LAN service has an owner and review date.

Read-only commands:

```bash
docker ps --format 'table {{.Names}}\t{{.Ports}}'
docker network ls
docker network inspect <network-name>
```

Host-level listener cross-check, if available:

```bash
ss -ltnp
# If process/PID details require privileges on the host, do not escalate just for the checklist.
# Fall back to listener-only output or lsof where available:
ss -ltn
lsof -nP -iTCP -sTCP:LISTEN
```

Good example:

```text
reverse-proxy: 0.0.0.0:443 intentional public entrypoint
app-backend: no published ports, reachable only on internal Docker network
admin-dashboard: 127.0.0.1:8080 local-only
```

Bad example:

```text
admin-dashboard: 0.0.0.0:8080 no auth note, no VPN requirement, no owner
```

## 3. Secrets

Goal: avoid secrets in images, compose files, repo history or published evidence.

- [ ] **[Critical]** No real secrets in committed compose files, Dockerfiles or docs.
- [ ] **[Critical]** Rotate exposed tokens immediately.
- [ ] **[High]** `.env` files are excluded by `.gitignore` and not copied into images.
- [ ] **[High]** Avoid passing secrets as command-line args visible in process listings.
- [ ] **[Medium]** Prefer Docker secrets, external secret managers, or local env injection depending on environment maturity.
- [ ] **[Hygiene]** Document where secrets live without documenting their values.

Read-only commands:

```bash
git status --short
git ls-files | grep -E '(^|/)\.env($|\.)|secret|token|credential|passwd|password'
grep -RIlE 'password|passwd|token|secret|api[_-]?key|jwt|bearer' . --exclude-dir=.git
```

Do not publish screenshots without reviewing them first; UI pages, reverse-proxy labels, paths and filenames can leak architecture or context. Treat `docker compose config` output as sensitive too because it may expand environment variables.

If a secret is found:

1. treat it as compromised;
2. rotate it;
3. remove it from working tree;
4. if committed, decide whether history rewrite is required before publishing.

Do **not** paste raw grep output into issues, chats or PRs. Even file names and paths can reveal architecture or customer context.

## 4. Dangerous privileges and escape paths

Goal: make dangerous capabilities intentional, rare and compensated.

Review containers using:

```text
--privileged
--net=host / network_mode: host
/var/run/docker.sock mount
broad host filesystem mounts
root user inside container
extra capabilities such as SYS_ADMIN, NET_ADMIN, SYS_PTRACE
host PID/IPC namespaces
```

Read-only commands:

```bash
docker inspect --format '{{.Name}} privileged={{.HostConfig.Privileged}} user={{.Config.User}} network={{.HostConfig.NetworkMode}} pid={{.HostConfig.PidMode}} ipc={{.HostConfig.IpcMode}} caps={{.HostConfig.CapAdd}}' $(docker ps -q)
docker inspect --format '{{.Name}} {{range .Mounts}}{{.Source}} -> {{.Destination}} rw={{.RW}}; {{end}}' $(docker ps -q)
```

Checklist:

- [ ] **[Critical]** `--privileged` has written justification and compensating controls.
- [ ] **[Critical]** Docker socket mount is avoided unless the container is explicitly a Docker controller.
- [ ] **[High]** `network_mode: host` is justified and documented.
- [ ] **[High]** Broad host mounts are replaced with narrow paths where possible.
- [ ] **[Medium]** Containers run as non-root where the image supports it.
- [ ] **[Medium]** Added Linux capabilities are minimal and documented.
- [ ] **[Hygiene]** Exceptions have owner and review date.

Exception notes:

- `--privileged`: acceptable only for specific low-level workloads where alternatives are not viable. Document why, what host access it enables, and how it is isolated.
- `root` inside container: sometimes required by vendor images. Prefer non-root images or explicit `user:` when supported.
- `host network`: useful for discovery, multicast, performance or certain homelab services. Compensate with host firewall and clear service ownership.
- `docker.sock`: equivalent to high control over the host Docker daemon. Treat as Critical. Prefer read-only proxies or scoped alternatives when possible.

## 5. Persistence, backups and restore

Goal: recover the service, not just preserve the compose file.

- [ ] **[High]** Identify persistent volumes and bind mounts.
- [ ] **[High]** Back up application data, databases and required config.
- [ ] **[High]** Test restore of at least one representative service.
- [ ] **[Medium]** Document restore order and dependencies.
- [ ] **[Medium]** Confirm backups are stored outside the same single disk/failure domain.
- [ ] **[Hygiene]** Keep a short restore runbook per critical stack.

Read-only commands:

```bash
docker volume ls
docker inspect --format '{{.Name}} {{range .Mounts}}{{.Type}} {{.Name}} {{.Source}} -> {{.Destination}}; {{end}}' $(docker ps -q)
find . -maxdepth 3 -iname '*backup*' -o -iname '*restore*' -o -iname 'README*'
```

Good restore note:

```text
service=db-backed-app
backup=data volume + postgres dump + compose file
restore order=database -> app -> reverse proxy label check
last tested=YYYY-MM-DD
```

Bad restore note:

```text
backup=compose.yml only
restore=not tested
```

## 6. Images, updates and rollback

Goal: stay patchable without breaking critical services blindly.

- [ ] **[Medium]** Prefer pinned tags over `latest` for important services.
- [ ] **[Medium]** Track image update cadence and upstream changelogs.
- [ ] **[High]** Keep rollback image/tag available before major upgrades.
- [ ] **[High]** Verify backup/snapshot before updating services with persistent data.
- [ ] **[Medium]** Document which services auto-update and which require manual review.
- [ ] **[Hygiene]** Remove unused images/containers only through an approved maintenance process.

Read-only commands:

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}'
docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.CreatedSince}}\t{{.Size}}'
# If using compose, keep output local/sanitized because variables may be expanded:
docker compose config
```

Good example:

```text
critical-db image=postgres:16.3 update=manual rollback=postgres:16.2 backup-before-upgrade=yes
```

Bad example:

```text
critical-db image=postgres:latest update=auto rollback=unknown backup=unknown
```

## Final validation

Before marking the review done:

- [ ] All Critical findings are fixed, isolated, or explicitly accepted with owner and review date.
- [ ] All High findings have a remediation plan.
- [ ] No raw command output with secrets or internal details is published.
- [ ] Every public/LAN-exposed service has documented intent.
- [ ] Every persistent service has backup and restore notes.
- [ ] Exceptions for `privileged`, `root`, `host network`, `docker.sock` are documented.
- [ ] Evidence is sanitized and stored locally.

Minimal local report template:

```text
review_date=YYYY-MM-DD
scope=<host-or-stack-sanitized>
critical=<count>
high=<count>
medium=<count>
hygiene=<count>
accepted_risks=<short sanitized list>
next_review=YYYY-MM-DD
```
