### 

### Installer le moteur Docker officiel
- Nettoyage et Préparation : `sudo apt update && sudo apt upgrade -y`
- Installation de Docker : `sudo apt install docker.io docker-compose-v2 -y`
- Configuration des Permissions : `sudo usermod -aG docker $USER`
	- Seul l'utilisateur `root` a le droit de manipuler Docker. 
		- Ajouter ton utilisateur actuel (`mgmt`) au groupe Docker
		- taper `newgrp docker`

### Audit et Nettoyage
**1. Corriger le cache Ubuntu (pour éliminer les erreurs 404) :**
``` bash
sudo apt update --fix-missing
```
**2. Vérifier si Docker est déjà présent :**
``` bash
docker --version
```
**3. Vérifier si le plugin Compose est déjà présent :**
``` bash
docker compose version
```