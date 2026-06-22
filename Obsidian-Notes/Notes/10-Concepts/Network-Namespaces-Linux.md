---

tags: [concept, linux, réseau, isolation]

liens: [OPNsense]

---
Une fonctionnalité avancée du noyau Linux : **les Network Namespaces (netns)** .
- Créais un "PC virtuel" dans ton Ubuntu
### Création de la machine cliente virtuelle (Namespace)
0. **Installe le client DHCP manuellement sur ton Ubuntu :**
``` 
sudo apt update && sudo apt install isc-dhcp-client -y
```
1. **Créer le conteneur réseau isolé (qu'on appellera "client_bureau") :**
``` bash
sudo ip netns add client_bureau
```
2. **Créer la carte réseau VLAN 20 (greffée sur ton interface physique ens33) :**
``` bash
sudo ip link add link ens33 name vlan20 type vlan id 20
```
3. **Déplacer cette carte VLAN dans l'espace isolé "client_bureau" :**
``` bash
sudo ip link set vlan20 netns client_bureau
```
### Phase 2 : Démarrage et requête DHCP
Maintenant, nous allons exécuter des commandes _à l'intérieur_ de ce namespace 
- (en utilisant `ip netns exec`).
1. **Allumer l'interface locale et l'interface VLAN :**
``` bash
sudo ip netns exec client_bureau ip link set lo up
sudo ip netns exec client_bureau ip link set vlan20 up
```
2. **Demander une adresse IP à ton OPNsense (via DHCP Kea) :**
``` bash
sudo ip netns exec client_bureau dhclient vlan20
```
    _(Patiente 2 ou 3 secondes, le temps que la négociation DHCP se fasse)._
### Le commutateur virtuel VMware (Promiscuous Mode)
Par défaut, le switch virtuel de VMware Workstation rejette le trafic contenant des adresses MAC qui ne correspondent pas exactement à la carte réseau virtuelle assignée à la VM.
- `vlan 20` est genere avec une nouvelle address MAC .
Nous pouvons forcer l'interface VLAN à utiliser la **même** adresse MAC que l'interface parente (`ens33`).
1. **Récupère la MAC de ens33 :**
``` bash
MAC=$(ip link show ens33 | awk '/ether/ {print $2}')
```
2. **Applique cette MAC au VLAN20 dans le namespace :**
``` bash
 sudo ip netns exec client_bureau ip link set vlan20 address $MAC
```
3. **Relance la requête DHCP :**
``` bash
   sudo ip netns exec client_bureau dhclient -v vlan20
```
    _(J'ai ajouté `-v` pour verbose, afin que nous puissions voir exactement les paquets partir)._
Si tu vois `DHCPDISCOVER` suivi de `DHCPOFFER`, le problème est résolu.
### Phase 3 : La Validation du Périmètre (Les 3 Tests)
**Test 1 : Validation de l'IP (Le DHCP a-t-il fonctionné ?)**
```  bash
sudo ip netns exec client_bureau ip a | grep vlan20
```
- IP entre 10.0.20.100 et 10.0.20.199
**Test 2 : Validation de la Porte de Sortie (Internet fonctionne-t-il ?)**
``` bash
sudo ip netns exec client_bureau ping -c 4 8.8.8.8
```
-  les paquets revenir. OPNsense route et NAT le trafic vers l'extérieur)
**Test 3 : Validation du Bouclier (Es-tu bien bloqué vers le Management ?) - LE PLUS IMPORTANT**
``` bash
sudo ip netns exec client_bureau ping -c 4 10.0.30.2
```

##### Validation du Filtrage (apres Pare-feu)
1. **L'accès Internet (La règle PASS et le NAT)**
``` bash
sudo ip netns exec client_bureau ping -c 4 8.8.8.8
```
_(Résultat attendu : Les paquets reviennent. Le Bureau 1 a accès au monde extérieur)._
**Test 2 : L'Isolation Zero Trust (La règle BLOCK) - LE TEST CRITIQUE**
``` bash
sudo ip netns exec client_bureau ping -c 4 10.0.30.2
```
### Phase 4 : Nettoyage de l'Environnement de Test
``` bash
sudo ip netns delete client_bureau
```