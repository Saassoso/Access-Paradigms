---

tags: [concept, réseau, vlan, l2]

liens: [OPNsense]

---
## Pourquoi segmenter ?
Un réseau plat (flat network) = un seul domaine de broadcast. Un attaquant qui compromet un endpoint peut atteindre tous les autres. La segmentation divise en zones avec contrôle d'accès entre elles.
## Comment fonctionne le tag 802.1Q
Chaque trame Ethernet reçoit un tag de 4 octets inséré dans l'en-tête, contenant le **VLAN ID** (0-4094).
```
Trame normale :
[MAC dest][MAC src][Type][Données][FCS]

Trame taggée 802.1Q :
[MAC dest][MAC src][0x8100][VLAN ID + priorité][Type][Données][FCS]
```
## Port trunk vs port access

| Type | Comportement | Usage |
|---|---|---|
| **Access** | Appartient à un seul VLAN, retire le tag | Côté endpoint (PC, VM) |
| **Trunk** | Transporte plusieurs VLANs avec leurs tags | Entre switches, switch → routeur |
## Inter-VLAN routing

Les VLANs sont isolés au niveau L2. Pour qu'ils communiquent, il faut du routage L3. Deux méthodes :
- **Router-on-a-stick** : une seule interface physique avec des sous-interfaces taggées (ce que fait OPNsense ici)
- **Switch L3** : le switch lui-même route entre VLANs

## Dans ce projet

Une seule interface physique `em1` sur [[../20 - Outils/OPNsense Configuration]] porte deux sous-interfaces VLAN :

| VLAN       | ID 802.1Q | Réseau IP      | Rôle                         |
| ---------- | --------- | -------------- | ---------------------------- |
| Management | **30**    | `10.0.30.0/29` | Docker Host, OPNsense WebGUI |
| Endpoints  | **20**    | `10.0.20.0/24` | VMs Windows                  |

> **Note** : L'ID 802.1Q (30) est différent de l'octet du sous-réseau IP (10). Ne pas confondre.

La VM Windows doit avoir le **tag VLAN 20 configuré dans les propriétés avancées de la carte réseau** (ou dans le fichier .vmx VMware) pour que les trames soient taggées correctement.

## Commandes de vérification

```bash
# Sur Docker Host Ubuntu — voir l'interface VLAN
ip a | grep vlan

# Test isolation depuis un namespace Linux
sudo ip netns exec client_bureau ping -c 4 10.0.30.2
# Résultat attendu : 100% packet loss

# Test accès internet depuis le VLAN 20
sudo ip netns exec client_bureau ping -c 4 8.8.8.8
# Résultat attendu : réponse
```