---

tags: [outil, réseau, ngfw, l7]

concepts: [Firewall Stateful]

parent: [[OPNsense]]

---
## Rôle

Plugin pour [[OPNsense Configuration]] qui transforme le firewall stateful L4 en **Next-Generation Firewall** capable d'inspecter le trafic au niveau applicatif (Layer 7).

## Ce qu'il ajoute

| OPNsense seul (L4) | Avec Zenarmor (L7) |
|---|---|
| `TCP port 443 autorisé` | `Netflix HTTPS streaming autorisé/bloqué` |
| Voit l'IP et le port | Voit l'application et la catégorie |
| Pas de DPI | Deep Packet Inspection |
| Pas de filtrage DNS | Filtrage par domaine/catégorie |
## Installation

System → Firmware → Plugins → chercher `os-sunnyvalley` → Install

## Dans ce projet

Status : installé, configuration de base en place.  
Usage principal : visibilité sur le trafic sortant du VLAN 20 (endpoints).