# Network Segmentation Checklist

A practical checklist for home networks with VLANs, IoT devices, homelab services, and guest access.

## 1. Define zones

- [ ] Trusted user devices.
- [ ] IoT devices.
- [ ] Guest devices.
- [ ] Homelab servers.
- [ ] Cameras/NVR.
- [ ] Work devices.
- [ ] Management plane.

For each zone, document:

```text
Name:
Purpose:
CIDR/VLAN:
Default gateway:
DNS:
Allowed inbound:
Allowed outbound:
Critical exceptions:
```

## 2. Default policy

- [ ] Avoid implicit any-to-any routing between VLANs.
- [ ] Permit only required flows.
- [ ] Keep management access limited to trusted admin devices.
- [ ] Do not allow IoT to initiate access to trusted user networks.
- [ ] Guest should not reach internal services.

## 3. DNS and discovery

- [ ] Decide how mDNS/SSDP is handled across VLANs.
- [ ] Document DNS resolvers per zone.
- [ ] Avoid leaking internal hostnames publicly.
- [ ] Monitor devices using hardcoded external DNS if relevant.

## 4. Home Assistant flows

Common required flows:

```text
HA → IoT devices
Trusted devices → HA UI/API
IoT devices → HA only when required
HA → DNS/NTP/MQTT
```

Avoid broad IoT → LAN access.

## 5. Logging and validation

- [ ] Log denied inter-VLAN attempts where practical.
- [ ] Test from each VLAN.
- [ ] Keep a simple allow matrix.
- [ ] Review firewall rules after adding new smart devices.

Validation examples:

```bash
ping <target>
nc -vz <host> <port>
nmap -Pn -p <ports> <target>
```

## 6. Change control

Before changing ACLs:

- [ ] Identify required source/destination.
- [ ] Identify exact ports/protocols.
- [ ] Define rollback.
- [ ] Test from a non-critical client first.
