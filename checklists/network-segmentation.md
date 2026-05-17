# Network Segmentation Checklist v0.2

A practical checklist for home networks with VLANs, IoT devices, homelab services, and guest access.

## Scope

Use this checklist to review whether a small home or homelab network has clear trust boundaries, intentional firewall rules, and enough documentation to change safely.

Covers:

- zone definitions and ownership;
- default inter-zone policy;
- management-plane isolation;
- IoT, guest, camera, and Home Assistant flows;
- DNS, discovery, and logging;
- validation and rollback notes.

This does not replace vendor-specific firewall guidance, wireless security review, ISP router hardening, endpoint hardening, or formal threat modeling.

## Assumptions

- You are reviewing a network you own or are authorized to administer.
- Example networks use documentation ranges or generic RFC1918 placeholders only.
- Rule exports, screenshots, device names, DHCP leases, VPN peers, and DNS logs may contain sensitive data.
- Do not make firewall or VLAN changes without a console path, backup, or rollback plan.

## Safety note - sanitize before sharing

Before publishing evidence, remove or generalize:

- real public IPs, ISP details, WAN hostnames, DDNS names, VPN endpoints, and serial numbers;
- internal hostnames, user names, device names, camera names, and room names;
- MAC addresses, DHCP leases, firewall rule IDs, and screenshots that reveal topology;
- exact management URLs, SSIDs, pre-shared keys, API tokens, and QR codes.

Sanitized example:

```text
GOOD: Trusted -> HA UI tcp/8123 allowed; IoT -> HA tcp/8123 denied except smart-hub-01.
BAD:  <real-user-device> -> <real-ddns-name> tcp/8123, <real-camera-name> MAC <redacted-mac>.
```

## Severity model

| Severity | Meaning | Typical action |
|---|---|---|
| Critical | Public or untrusted access to management, cameras, identity, or unsafe automations | Isolate or block immediately |
| High | Broad inter-zone access or weak management boundary | Prioritize remediation |
| Medium | Missing documentation, logging, or overly broad exception | Schedule fix |
| Hygiene | Naming, review dates, consistency, or cleanup | Improve when practical |

## Quick audit - 15 minutes

Use this first to decide whether the segmentation model is trustworthy enough for routine maintenance.

| Check | Severity | Look for | Red flags |
|---|---:|---|---|
| Management access | Critical | Firewall, router, AP, switch, NAS, Proxmox, HA admin UIs | Reachable from Guest, IoT, Cameras, or WAN |
| Default inter-zone policy | High | LAN/VLAN firewall rules | Any-to-any rules between trusted and untrusted zones |
| Guest isolation | High | Guest SSID/VLAN rules | Guest can reach RFC1918 internal services |
| IoT access | High | IoT outbound and inbound rules | IoT can initiate broad access to Trusted or Management |
| Camera/NVR access | High | Camera VLAN rules | Cameras can initiate internet or Trusted access without reason |
| DNS and NTP | Medium | Resolver and time rules | Devices bypass local DNS policy unexpectedly |
| Logging | Medium | Deny logs for critical boundaries | No evidence when a risky rule is hit |
| Documentation | Hygiene | Zone owner, purpose, review date | Exceptions with no owner or expiry |

> If Guest, IoT, or Cameras can reach management interfaces, stop treating the network as segmented until the rule is removed, justified, or compensated.

## 1. Define zones

- [ ] **[High]** Trusted user devices are separated from untrusted or semi-trusted devices.
- [ ] **[High]** IoT devices have their own zone or clearly documented compensating controls.
- [ ] **[High]** Guest devices cannot reach internal services.
- [ ] **[High]** Homelab servers are separated from user devices where practical.
- [ ] **[High]** Cameras/NVR are isolated from Trusted and Management zones.
- [ ] **[Critical]** Management plane access is limited to trusted admin devices.
- [ ] **[Medium]** Work devices have documented access boundaries if they touch personal infrastructure.

For each zone, document:

```text
Name:
Purpose:
CIDR/VLAN: <sanitized or placeholder>
Default gateway:
DNS:
Allowed inbound:
Allowed outbound:
Critical exceptions:
Owner:
Review date:
```

## 2. Default policy

- [ ] **[High]** Avoid implicit any-to-any routing between VLANs.
- [ ] **[High]** Permit only required flows.
- [ ] **[Critical]** Keep management access limited to trusted admin devices.
- [ ] **[High]** Do not allow IoT to initiate access to trusted user networks.
- [ ] **[High]** Guest cannot reach internal services.
- [ ] **[High]** Cameras cannot initiate access to Trusted or Management zones.
- [ ] **[Medium]** Allow rules are narrower than entire RFC1918 ranges unless intentionally documented.
- [ ] **[Medium]** Temporary exceptions have an owner, purpose, and expiry/review date.

Minimum allow-matrix:

| Source | Destination | Typical policy | Notes |
|---|---|---|---|
| Trusted | Management | Allow selected admin protocols | Admin devices only |
| Trusted | Homelab | Allow required app ports | Avoid broad management by default |
| Trusted | IoT | Allow only needed control paths | Prefer app/controller mediated access |
| Guest | Any internal zone | Deny | Internet-only unless explicitly approved |
| IoT | Trusted | Deny | Exceptions should be device-specific |
| IoT | Home Assistant | Allow selected ports if required | Prefer HA-initiated flows when possible |
| Cameras | NVR | Allow camera protocols | Keep camera internet access intentional |
| Homelab | Internet | Allow required updates/services | Watch for unexpected outbound services |

## 3. DNS and discovery

- [ ] **[Medium]** Decide how mDNS/SSDP is handled across VLANs.
- [ ] **[Medium]** Document DNS resolvers per zone.
- [ ] **[High]** Avoid leaking internal hostnames publicly.
- [ ] **[Medium]** Monitor devices using hardcoded external DNS if relevant.
- [ ] **[Medium]** Decide whether IoT DNS should be filtered, redirected, or only observed.
- [ ] **[Hygiene]** Document exceptions for devices that need vendor cloud discovery.

## 4. Home Assistant flows

Common required flows:

```text
HA -> IoT devices
Trusted devices -> HA UI/API
IoT devices -> HA only when required
HA -> DNS/NTP/MQTT
```

Avoid broad IoT -> LAN access.

Review Home Assistant separately if it controls locks, alarms, climate, energy cutoffs, cameras, or presence-based access. Segmentation mistakes can become safety and availability issues, not just network hygiene.

## 5. Logging and validation

- [ ] **[Medium]** Log denied inter-VLAN attempts where practical.
- [ ] **[Medium]** Test from each VLAN or representative SSID.
- [ ] **[Medium]** Keep a simple allow matrix.
- [ ] **[Medium]** Review firewall rules after adding new smart devices.
- [ ] **[High]** Validate that management interfaces are unreachable from Guest, IoT, and Cameras.
- [ ] **[High]** Validate that exposed services match the intended audience: local-only, LAN-only, VPN-only, or public.

Validation examples:

```bash
ping <target>
nc -vz <host> <port>
nmap -Pn -p <ports> <target>
```

Evidence format:

```text
source_zone=<zone> destination=<zone-or-service> protocol=<tcp/udp/icmp> port=<port> result=<allowed|denied> expected=<yes|no> note=<sanitized>
```

## 6. Change control

Before changing ACLs:

- [ ] **[High]** Identify required source/destination.
- [ ] **[High]** Identify exact ports/protocols.
- [ ] **[High]** Define rollback.
- [ ] **[Medium]** Test from a non-critical client first.
- [ ] **[Medium]** Capture before/after evidence.
- [ ] **[Hygiene]** Record the reason, owner, and review date.

Change note template:

```text
Change:
Reason:
Owner:
Source zone:
Destination:
Ports/protocols:
Expected behavior:
Rollback:
Validation:
Review date:
```
