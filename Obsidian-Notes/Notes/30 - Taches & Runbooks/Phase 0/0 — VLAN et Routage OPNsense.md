---

tags: [tâche, phase-0, réseau, opnsense]

outils: [OPNsense, Zenarmor]

concepts: [VLAN Trunking 802.1Q, Firewall Stateful]

statut: ✅ complété

---
## Contexte

Mise en place de la segmentation réseau zero-trust.  
Deux VLANs sur trunk 802.1Q (interface em1 d'OPNsense).

## Étapes complétées

### 1. Création VM OPNsense (VMware)
- 2 interfaces : NAT (em0/WAN) + LAN Segment (em1/Trunk)
- OVA ou ISO OPNsense installé

### 2. Configuration initiale console OPNsense
```
WAN → em0 (DHCP)
LAN → em1 (Trunk)

LAN IP : 10.0.30.1/29
DHCP sur LAN : NON (réseau Management — accès restreint)
```

Décocher dans Setup Wizard :
- Block RFC1918 Private Networks (WAN = NAT VMware = IP privée)
- Optimize for Multiwan

### 3. Création VLAN 20 (Endpoints)

OPNsense : Interfaces → Devices → VLAN → +
```
Parent : em1
Tag : 20
Description : VLAN20_Bureau
```

Interfaces → Assignments → Ajouter VLAN20_Bureau → activer  
IP : `10.0.20.1/24`, DHCP Kea activé pool `10.0.20.100-199`

### 4. VLAN 30 (Management) = interface em1 native

L'interface LAN em1 native est renommée VLAN30_Management.  
IP déjà configurée : `10.0.30.1/29`

> **Note cohérence** : VLAN ID 802.1Q = 30, réseau IP = 10.0.30.0/29.  
> L'octet du réseau (10) ≠ l'ID VLAN (30). Ne pas confondre.

### 5. Règles firewall sur VLAN20_Bureau

Firewall → Rules → VLAN20_Bureau

```
Règle 1 (BLOCK) :
  Action : Block
  Source : VLAN20_Bureau net
  Destination : LAN net (10.0.30.0/29)
  Description : ISOLATION zero-trust — Endpoints → Management

Règle 2 (PASS) :
  Action : Pass
  Source : VLAN20_Bureau net
  Destination : any
  Description : Accès internet
```

### 6. Zenarmor

System → Firmware → Plugins → `os-sunnyvalley` → Install  
Configuration de base appliquée.

## Validation 

| Test                 | Commande             | Résultat           |
| -------------------- | -------------------- | ------------------ |
| DHCP VLAN 20         | `ip a` sur namespace | IP 10.0.20.100-199 |
| Internet             | `ping 8.8.8.8`       | Réponse            |
| Isolation Management | `ping 10.0.30.2`     | 100% packet loss   |

Voir [[Lab - Test isolation VLAN]] pour les détails des tests.

## Docker Host — configuration réseau

Voir [[netplan]] pour la configuration IP statique du Docker Host.  
Voir [[Runbook - Accès OPNsense WebGUI]] pour l'accès à la WebGUI.
