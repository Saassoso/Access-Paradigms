- OPNsense est plus ouvert aux automatisations modernes requises.
### Topologie Hyperviseur (Couche 2)
OPNsense aura exactement **DEUX** interfaces virtuelles dans VMware.
- **WAN :** Connecté au réseau NAT.
- **LAN :** Connecté à un LAN Segment unique. Ce lien agira comme un Trunk 802.1Q.
### Topologie Logique (Couche 3 - VLANs sur OPNsense)
Sur cette interface LAN unique, on vas instancier des réseaux virtuels.

On configure un Trunk 802.1Q (une seule interface) et on y tague les différents sous-réseaux (VLANs)
- **VLAN Management :** **(ID 30) :** `10.0.10.0/29`
	- Ce segment n'hébergera que ton infrastructure vitale (Interface OPNsense, IP de la VM Docker Host , IP de management de l'hyperviseur ). 
- **VLAN Endpoints :** **(ID 20) :** `10.0.20.0/24` (DHCP: 100-199)
	- Ce segment accueillera ta VM Windows 11 cible et les potentiels futurs postes clients. Un sous-réseau standard est ici pour allouer le pool DHCP.
### Configuration OPNsense 
on chosis em0, l'interface WAN 
- et N : pour refuser les LAGGs.
- et N : Do you want to configure VLANs now?
![](Notes/30%20-%20Taches%20&%20Runbooks/images/1-OPNsense.png)
Assign interfaces : name
- Trunk : Lan Segment ; em1 ;
- Wan : NAT ; em0 ; 

Set interface IP address 
- **WAN**. Configure l'IPv4 en **DHCP**. Ne configure pas d'IPv6.
- **LAN**.
	- Configure l'adresse IPv4 statique : `10.0.10.1`
	- Masque de sous-réseau (Subnet bit count) : `29`
	- Appuie sur Entrée pour ne pas définir de passerelle (Gateway) sur le LAN.
	- Ne configure pas d'IPv6.
	- Désactive le serveur DHCP sur le LAN pour le moment (nous n'en voulons pas sur le réseau de Management restreint).
	- Ne modifie pas le protocole WebGUI (garde HTTPS).

### Access WebGUI
Utiliser port forwarding over docker-host , ssh tunnel connection over terminus pour faire rebondir ton navigateur local :
![](Notes/30%20-%20Taches%20&%20Runbooks/images/0-OPNsense.png)
#### Configuration Reseau et du Tunnel dans [[Termius]]
- [[Notes/40 - Reference/netplan]] : to configure the network of the docker host
- `root` et le mot de passe `opnsense`
### Assistant Initial
- **Connexion :** Accède à `https://127.0.0.1:8444`. 
	- On Accepte l'avertissement de sécurité lié au certificat auto-signé.
- **Identifiants :** Connecte-toi avec `root` et le mot de passe `opnsense`. 
	- L'assistant de configuration initiale va se lancer automatiquement.
- **General Information :** * Hostname : `opnsense`
    - Domain : `charif-labs.tech`
    - Primary DNS Server : `1.1.1.1`
- **Time Server Information :** GMT+1.
- **Configure WAN Interface (Étape Critique) :**
    - Laisse le type sur **DHCP**.
    - Fais défiler tout en bas de la page.
    - **DÉCOCHE obligatoirement "Block RFC1918 Private Networks"** et **"Block bogon networks"**
	    - WAN est connecté au réseau NAT de VMware (`192.168.138.x`), qui est une IP privée. 
	    - Si on laisses ces cases cochées, OPNsense bloquera tout le trafic sortant.
- **Configure LAN Interface :**  `10.0.10.1` et le masque `255.255.255.248` (qui correspond à ton `/29`).
- **Set Root Password :** 
- **Optimize for Multiwan : DÉCOCHE cette case.** Ton pare-feu ne possède qu'une seule interface de sortie (le WAN NAT de VMware).
- **Décoche "Automatic DHCP/DNS registration"** (c'est ce qui bloque ton clic).
- **Reload :** Applique les paramètres.

### VLANs
#### Creation du Tag 802.1Q : 20
![](Notes/30%20-%20Taches%20&%20Runbooks/images/2-OPNsense.png)
**Devices** -> **VLAN** -> **+** 
##### Renseigne _exclusivement_ les champs
- **Parent interface :** Sélectionne l'interface Trunk, c'est-à-dire **`em1`** (elle sera probablement affichée sous la forme `em1 (adresse_mac) [LAN]`). Ne sélectionne surtout pas le WAN (`em0`).
- **VLAN tag :** `20`
- **VLAN priority :** Laisse la valeur par défaut (`0`).
- **Description :** `VLAN20_Bureau1`
Then Click **Apply**
##### Assignation Logique (Couche L3)
Interfaces > Assignments -> **New interface**. -> VLAN -> + -> OPTS
![456](Notes/30%20-%20Taches%20&%20Runbooks/images/OPNsense.png)
##### Configuration IP (Passerelle du VLAN)
- Dans le menu de gauche, va dans **Interfaces** et clique sur **[VLAN20_Bureau]**.
- **Enable :** Coche la case "Enable Interface" (sans cela, l'interface reste éteinte).
- **IPv4 Configuration Type :** Sélectionne **Static IPv4**.
- Descends jusqu'à la section _Static IPv4 configuration_ :
    - **IPv4 address :** Saisis `10.0.20.1`
    - **Subnet mask :** Sélectionne `24` dans le menu déroulant (ce qui correspond à ton masque `255.255.255.0`).
- Laisse l'IPv6 désactivé (None) et ne touche pas à la passerelle (Upstream Gateway).
- Clique sur **Save** tout en bas, puis clique impérativement sur le bouton **Apply Changes** qui apparaîtra en haut.
##### Activation du Serveur DHCP
Le pare-feu possède l'adresse `10.0.20.1` sur ce réseau, il peut distribuer des IPs aux machines du Bureau 1.
- **Services > Kea DHCP **> Ker DHCPv4** : Enable 
- **Enable DHCP server on the [VLAN20_Bureau] interface**.
- Dans l'onglet **Subnets** : Clique sur **+** pour ajouter ton réseau.
    - **Subnet** : `10.0.20.0/24`
    - **Pools** : `10.0.20.100 - 10.0.20.199`
- Dans l'onglet **Options** (toujours dans la création du Subnet) :
    - Cherche le code `3` (Routers) et mets `10.0.20.1`.
    - Cherche le code `6` (Domain Name Servers) et mets `1.1.1.1`.
### Routage dans pc 
![](Notes/30%20-%20Taches%20&%20Runbooks/images/OPNsense-1.png)
#### Test Vlans
##### Tagging 802.1Q (Dans l'OS Client)
- `ip a` (ex: `ens33`).
- Crée l'interface VLAN 20 avec cette commande (remplace `ens33` par ton interface) : `sudo nmcli connection add type vlan con-name VLAN20 dev ens33 id 20 ipv4.method auto`
- Active la connexion : `sudo nmcli connection up VLAN20`
`/etc/netplan/50-cloud-init.yaml`

``` yaml
network:
  version: 2
  ethernets:
    ens33: {}
  vlans:
    vlan20:
      id: 20
      link: ens33
      dhcp4: true
```
##### Attribution IP (DHCP Kea)
`ip a | grep vlan20` -> adresse IP assignée entre `10.0.20.100` et `10.0.20.199`. 
	Cela valide la couche 2, la couche 3 et ton service DHCP.
##### La Porte de Sortie (Internet)
`ping -c 4 8.8.8.8` -> Les paquets doivent revenir. Cela valide ta règle de pare-feu "ALLOW" et le NAT sortant d'OPNsense.
##### Bouclier (Isolation du Management)
`ping -c 4 10.0.10.2` (L'IP de ton Ubuntu Docker Host)
	100% packet loss
#### [[Notes/10 - Concepts/Network Namespaces Linux]]
### Configuration Firewall
#### Règle d'Isolation
**Firewall > Rules > [VLAN20_Bureau]**.
- **Action :** `Block`
- **Interface :** `[VLAN20_Bureau]`
- **Direction :** `in`
- **Protocol :** `any`
- **Source :** `VLAN20_Bureau net` 
- **Destination :**  `LAN net` 
- **Description :** `ISOLATION : Bloquer Bureau vers Management`
#### Règle d'Accès Internet
**Firewall > Rules > [VLAN20_Bureau]**.
- **Action :** `Pass`
- **Interface :** `[VLAN20_Bureau]`
- **Direction :** `in`
- **Protocol :** `any`
- **Source :** `VLAN20_Bureau net` 
- **Destination :**  `any` 
- **Description :** `ISOLATION : Bloquer Bureau vers Management`
#### Test Firewall
