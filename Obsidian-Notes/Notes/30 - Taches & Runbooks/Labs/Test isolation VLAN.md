---

tags: [lab, réseau, validation, vlan]

outils: [OPNsense]

concepts: [Network Namespaces Linux, VLAN Trunking 802.1Q]

statut: ✅ validé

---
## Objectif

Valider les règles firewall OPNsense sans avoir besoin d'une VM Windows dédiée.  
Utilise [[Network Namespaces Linux]] depuis le Docker Host pour simuler un client VLAN 20.

## Prérequis

```bash
sudo apt install isc-dhcp-client -y
```

## Étapes

```bash
# 1. Créer le namespace + interface VLAN 20
sudo ip netns add client_bureau
sudo ip link add link ens33 name vlan20 type vlan id 20

# 2. Forcer la même MAC (nécessaire avec VMware)
MAC=$(ip link show ens33 | awk '/ether/ {print $2}')
sudo ip link set vlan20 netns client_bureau
sudo ip netns exec client_bureau ip link set vlan20 address $MAC

# 3. Activer les interfaces
sudo ip netns exec client_bureau ip link set lo up
sudo ip netns exec client_bureau ip link set vlan20 up

# 4. Obtenir une IP DHCP
sudo ip netns exec client_bureau dhclient -v vlan20
```

## Les 3 tests de validation

```bash
# Test 1 — IP dans le bon range ?
sudo ip netns exec client_bureau ip a | grep vlan20
#  Attendu : 10.0.20.100 à 10.0.20.199
```

```bash
# Test 2 — Internet accessible ?
sudo ip netns exec client_bureau ping -c 4 8.8.8.8
#  Attendu : réponse (règle PASS + NAT fonctionne)
```

```bash
# Test 3 — Management ISOLÉ ? (TEST CRITIQUE)
sudo ip netns exec client_bureau ping -c 4 10.0.10.2
#  Attendu : 100% packet loss (règle BLOCK fonctionne)
```
## Résultats

| Test                 | Attendu      | Obtenu       | Statut |
| -------------------- | ------------ | ------------ | ------ |
| DHCP VLAN 20         | IP 10.0.20.x | 10.0.20.100  | Done   |
| Internet             | Réponse ping | 4/4 réponses | Done   |
| Isolation Management | 0% réponse   | 100% loss    | Done   |
## Nettoyage
```bash
sudo ip netns delete client_bureau
```
