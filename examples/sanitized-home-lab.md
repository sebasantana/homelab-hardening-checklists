# Sanitized Homelab Example

This is a fictional example. Do not publish real internal diagrams, public IPs,
customer data, or secrets. The CIDRs use documentation ranges and should not be
copied as a literal production design.

## Network zones

| Zone | Purpose | Example documentation CIDR |
|---|---|---|
| HomeLAN | Trusted user devices | 192.0.2.0/28 |
| IoT | Smart devices | 192.0.2.16/28 |
| Guest | Visitors | 192.0.2.32/28 |
| Homelab | Servers and VMs | 192.0.2.48/28 |
| Cameras | Video devices | 192.0.2.64/28 |

## Policy example

- IoT can reach Home Assistant only.
- Guest cannot reach internal services.
- Homelab management is restricted to admin devices.
- Cameras cannot initiate connections to trusted zones.
