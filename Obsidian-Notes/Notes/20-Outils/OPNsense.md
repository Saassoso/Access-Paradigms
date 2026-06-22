---

tags: [outil, réseau, firewall]

concepts: [VLAN Trunking 802.1Q, Firewall Stateful]

---
## Rôle dans ce projet

Firewall de périmètre. Gère le routage inter-VLAN, les règles d'isolation zero-trust, et le NAT sortant. Interface physique em1 en trunk 802.1Q portant VLAN 20 et VLAN 30.

Comprend [[VLAN Trunking 802.1Q]] et [[Firewall Stateful]] avant de configurer.

## Interfaces

| Interface       | em     | VLAN ID | Réseau       | Rôle                  |
| --------------- | ------ | ------- | ------------ | --------------------- |
| WAN             | em0    | —       | NAT VMware   | Sortie internet       |
| LAN (trunk)     | em1    | —       | —            | Trunk 802.1Q          |
| VLAN Management | em1.30 | 30      | 10.0.30.0/29 | Docker Host, OPNsense |
| VLAN Endpoints  | em1.20 | 20      | 10.0.20.0/24 | VMs Windows           |

## Configuration initiale critique

Dans l'assistant Setup :
- WAN : DHCP (réseau NAT VMware = IP privée)
- **Décocher "Block RFC1918 Private Networks"** — sinon OPNsense bloque son propre WAN
- **Décocher "Optimize for Multiwan"** — une seule interface WAN
- **Décocher "Automatic DHCP/DNS registration"** si ça bloque

## DHCP Kea sur VLAN 20

Services → Kea DHCP → Kea DHCPv4 → Enable  
Subnet : `10.0.20.0/24`  
Pool : `10.0.20.100 - 10.0.20.199`  
Options : Router = `10.0.20.1`, DNS = `1.1.1.1`

## Règles firewall (interface VLAN20_Bureau)

Ordre d'évaluation — **la règle BLOCK doit être en premier** :

```
1. BLOCK   — src: VLAN20 net  → dst: LAN net      (isolation zero-trust)
2. PASS    — src: VLAN20 net  → dst: any           (internet)
```

## Accès WebGUI

OPNsense WebGUI tourne sur `https://10.0.30.1:443` (VLAN 30). Non accessible directement depuis le PC physique.  
→ Utiliser le [[Runbook - Accès OPNsense WebGUI]] (SSH tunnel via Termius).

## Zenarmor

Plugin NGFW Layer 7. S'installe depuis System → Firmware → Plugins → `os-sunnyvalley`.  
Voir [[Zenarmor]].
