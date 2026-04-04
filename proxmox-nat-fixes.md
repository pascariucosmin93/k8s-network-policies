# Proxmox NAT Fixes (2026-04-03)

## Problema

Doua subnet-uri nu aveau acces la internet din cauza unor reguli SNAT lipsa sau gresite pe Proxmox (10.90.90.13 / 10.13.13.1).

---

## 1. 10.13.13.0/24 — Regula SNAT lipsea complet

**Simptom:** VM-urile din subnet-ul 10.13.13.x ajungeau la gateway (10.13.13.1) dar pachetele ieseau spre internet cu IP sursa privat, fara NAT.

**Fix aplicat (runtime):**
```bash
iptables -t nat -A POSTROUTING -s 10.13.13.0/24 -o vmbr0 -j SNAT --to-source 10.90.90.13
```

**Fix persistent** — adauga in `/etc/network/interfaces` la sectiunea `vmbr0`:
```
post-up   iptables -t nat -A POSTROUTING -s 10.13.13.0/24 -o vmbr0 -j SNAT --to-source 10.90.90.13
post-down iptables -t nat -D POSTROUTING -s 10.13.13.0/24 -o vmbr0 -j SNAT --to-source 10.90.90.13
```

---

## 2. 192.168.70.0/24 — Regula SNAT cu IP inexistent

**Simptom:** Exista o regula SNAT pentru 192.168.70.0/24 dar facea SNAT spre `10.88.88.230` — un IP care nu mai era configurat pe nicio interfata a Proxmox-ului.

**Fix aplicat (runtime):**
```bash
# Sterge regula stricata
iptables -t nat -D POSTROUTING -s 192.168.70.0/24 -o vmbr0 -j SNAT --to-source 10.88.88.230

# Adauga regula corecta
iptables -t nat -A POSTROUTING -s 192.168.70.0/24 -o vmbr0 -j SNAT --to-source 10.90.90.13
```

**Fix persistent** — adauga in `/etc/network/interfaces` la sectiunea `vmbr0`:
```
post-up   iptables -t nat -A POSTROUTING -s 192.168.70.0/24 -o vmbr0 -j SNAT --to-source 10.90.90.13
post-down iptables -t nat -D POSTROUTING -s 192.168.70.0/24 -o vmbr0 -j SNAT --to-source 10.90.90.13
```

---

## Starea actuala a regulilor NAT (POSTROUTING)

| # | Subnet sursa     | Interfata | SNAT target    | Status       |
|---|-----------------|-----------|----------------|--------------|
| 1 | 20.1.1.0/24     | vmbr0     | 10.88.88.230   | ⚠️ IP inexistent |
| 2 | 20.1.1.0/24     | vmbr0     | 10.90.90.13    | ✓ OK (duplicat) |
| 3 | 192.168.70.0/24 | vmbr0     | 10.90.90.13    | ✓ OK (activa) |
| 8 | 10.13.13.0/24   | vmbr0     | 10.90.90.13    | ✓ OK (adaugata azi) |

> ⚠️ Regulile cu `10.88.88.230` si duplicatele pentru `20.1.1.0/24` ar trebui curatate.

---

## Curatare reguli redundante (optional)

```bash
# Sterge toate regulile vechi cu IP inexistent sau duplicate
iptables -t nat -D POSTROUTING -s 20.1.1.0/24 -o vmbr0 -j SNAT --to-source 10.88.88.230
iptables -t nat -D POSTROUTING -s 20.1.1.0/24 -o vmbr0 -j SNAT --to-source 10.90.90.13
iptables -t nat -D POSTROUTING -s 20.1.1.0/24 -o vmbr0 -j SNAT --to-source 10.90.90.13
iptables -t nat -D POSTROUTING -s 192.168.70.0/24 -o vmbr0 -j SNAT --to-source 10.90.90.13
iptables -t nat -D POSTROUTING -s 192.168.70.0/24 -o vmbr0 -j SNAT --to-source 10.90.90.13

# Pastreaza doar regulile necesare
iptables -t nat -A POSTROUTING -s 192.168.70.0/24 -o vmbr0 -j SNAT --to-source 10.90.90.13
iptables -t nat -A POSTROUTING -s 10.13.13.0/24   -o vmbr0 -j SNAT --to-source 10.90.90.13
```

---

## Note arhitectura

- `vmbr0` (10.90.90.13/24) — bridge spre internet, default gateway via 10.90.90.1
- `vmbr1` (10.20.20.230/24) — bridge intern
- `vnet1` (10.13.13.1/24) — retea management VM-uri
- `vnet2` (192.168.70.2/24) — retea interna cluster Kubernetes
