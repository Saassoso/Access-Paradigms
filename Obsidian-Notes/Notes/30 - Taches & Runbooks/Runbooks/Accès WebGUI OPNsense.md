---

tags: [runbook, opnsense, accès, ssh]

outils: [OPNsense]

déclencheur: Besoin d'accéder à la WebGUI OPNsense depuis le PC physique

---
## Problème
OPNsense WebGUI (`https://10.0.30.1:443`) est sur le VLAN 30 (Management).  
Le PC physique n'est pas sur ce réseau.
## Solution
SSH tunnel via le Docker Host (qui lui est sur VLAN 30).
## Étapes avec Termius
### 1. Créer l'hôte Docker Host dans Termius
```
Label : Docker Host
Host : 10.0.30.2
Port : 22
Username : mgmt (ou ton user Ubuntu)
Auth : clé SSH sovereign_ed25519
```
### 2. Créer la règle Port Forwarding

```
Type : Local
Local port : 8444
Destination address : 10.0.30.1
Destination port : 443
Select Host : Docker Host
```
### 3. Activer le tunnel
Double-clic sur la règle → statut "Connected"
### 4. Accéder à OPNsense
Navigateur → `https://127.0.0.1:8444`  
Accepter l'avertissement certificat auto-signé.  
Login : `root` / mot de passe OPNsense
## Version CLI (sans Termius)
```bash
ssh -L 8444:10.0.30.1:443 mgmt@10.0.30.2 -N -f
# -L : local forwarding
# -N : pas de commande (juste le tunnel)
# -f : passer en background
```
## Fermer le tunnel
Termius : déconnecter la règle  
CLI : `kill $(lsof -ti:8444)`
