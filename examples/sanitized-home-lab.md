# Sanitized Homelab Example

This is a fictional example. Do not publish real internal diagrams, public IPs, customer data, or secrets.

## Network zones

| Zone | Purpose | Example CIDR |
|---|---|---|
| HomeLAN | Trusted user devices | 10.0.10.0/24 |
| IoT | Smart devices | 10.0.20.0/24 |
| Guest | Visitors | 10.0.30.0/24 |
| Homelab | Servers and VMs | 10.0.40.0/24 |
| Cameras | Video devices | 10.0.50.0/24 |

## Policy example

- IoT can reach Home Assistant only.
- Guest cannot reach internal services.
- Homelab management is restricted to admin devices.
- Cameras cannot initiate connections to trusted zones.
