# Proxmox NAT Fixes (2026-04-03)

## Problem

Two subnets did not have internet access because of missing or incorrect SNAT rules on Proxmox (10.90.90.13 / 10.13.13.1).

---

## 1. 10.13.13.0/24 - SNAT rule was completely missing

**Symptom:** VMs in subnet 10.13.13.x could reach the gateway (10.13.13.1), but packets were leaving to the internet with private source IPs, without NAT.

**Applied fix (runtime):**
```bash
iptables -t nat -A POSTROUTING -s 10.13.13.0/24 -o vmbr0 -j SNAT --to-source 10.90.90.13
```

**Persistent fix** - add this to `/etc/network/interfaces` in the `vmbr0` section:
```
post-up   iptables -t nat -A POSTROUTING -s 10.13.13.0/24 -o vmbr0 -j SNAT --to-source 10.90.90.13
post-down iptables -t nat -D POSTROUTING -s 10.13.13.0/24 -o vmbr0 -j SNAT --to-source 10.90.90.13
```

---

## 2. 192.168.70.0/24 - SNAT rule with non-existent IP

**Symptom:** There was an SNAT rule for 192.168.70.0/24, but it was translating to `10.88.88.230` - an IP that was no longer configured on any Proxmox interface.

**Applied fix (runtime):**
```bash
# Remove broken rule
iptables -t nat -D POSTROUTING -s 192.168.70.0/24 -o vmbr0 -j SNAT --to-source 10.88.88.230

# Add correct rule
iptables -t nat -A POSTROUTING -s 192.168.70.0/24 -o vmbr0 -j SNAT --to-source 10.90.90.13
```

**Persistent fix** - add this to `/etc/network/interfaces` in the `vmbr0` section:
```
post-up   iptables -t nat -A POSTROUTING -s 192.168.70.0/24 -o vmbr0 -j SNAT --to-source 10.90.90.13
post-down iptables -t nat -D POSTROUTING -s 192.168.70.0/24 -o vmbr0 -j SNAT --to-source 10.90.90.13
```

---

## Current state of NAT rules (POSTROUTING)

| # | Source subnet    | Interface | SNAT target    | Status       |
|---|-----------------|-----------|----------------|--------------|
| 1 | 20.1.1.0/24     | vmbr0     | 10.88.88.230   | WARNING: non-existent IP |
| 2 | 20.1.1.0/24     | vmbr0     | 10.90.90.13    | OK (duplicate) |
| 3 | 192.168.70.0/24 | vmbr0     | 10.90.90.13    | OK (active) |
| 8 | 10.13.13.0/24   | vmbr0     | 10.90.90.13    | OK (added today) |

> WARNING: Rules with `10.88.88.230` and duplicate entries for `20.1.1.0/24` should be cleaned up.

---

## Cleanup of redundant rules (optional)

```bash
# Remove all old rules with non-existent IPs or duplicates
iptables -t nat -D POSTROUTING -s 20.1.1.0/24 -o vmbr0 -j SNAT --to-source 10.88.88.230
iptables -t nat -D POSTROUTING -s 20.1.1.0/24 -o vmbr0 -j SNAT --to-source 10.90.90.13
iptables -t nat -D POSTROUTING -s 20.1.1.0/24 -o vmbr0 -j SNAT --to-source 10.90.90.13
iptables -t nat -D POSTROUTING -s 192.168.70.0/24 -o vmbr0 -j SNAT --to-source 10.90.90.13
iptables -t nat -D POSTROUTING -s 192.168.70.0/24 -o vmbr0 -j SNAT --to-source 10.90.90.13

# Keep only required rules
iptables -t nat -A POSTROUTING -s 192.168.70.0/24 -o vmbr0 -j SNAT --to-source 10.90.90.13
iptables -t nat -A POSTROUTING -s 10.13.13.0/24   -o vmbr0 -j SNAT --to-source 10.90.90.13
```

---

## Architecture notes

- `vmbr0` (10.90.90.13/24) - internet-facing bridge, default gateway via 10.90.90.1
- `vmbr1` (10.20.20.230/24) - internal bridge
- `vnet1` (10.13.13.1/24) - VM management network
- `vnet2` (192.168.70.2/24) - internal Kubernetes cluster network
