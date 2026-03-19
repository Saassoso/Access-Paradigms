> Tout ce que tu dois maîtriser AVANT de commencer chaque phase. Avec le "pourquoi" pour chaque compétence.

---
## Comment utiliser ce document
Chaque compétence est annotée avec :
- **Quand tu en as besoin** : la phase concernée
- **Pourquoi** : ce qui casse sans cette compétence
- **Niveau requis** : 🟡 Notions | 🟠 Pratique | 🔴 Maîtrise
---
## Phase 0 — Fondations
### VLAN & Routage L2/L3 — 🔴 Maîtrise requise
**Pourquoi** : [[OPNsense]] est le cœur du projet. Sans comprendre les VLANs, tu ne peux pas isoler les zones réseau ni debugger les problèmes de connectivité entre agents et serveurs.

**Ce que tu dois savoir** :
- Trunk 802.1Q vs access port : une interface physique peut porter plusieurs VLANs tagués
- Le tag VLAN voyage dans l'en-tête Ethernet (4 octets, champ 802.1Q)
- OPNsense instancie des sous-interfaces VLAN sur l'interface trunk
- Routage inter-VLAN = OPNsense fait le routage L3 entre les VLANs
- Firewall stateful : une règle `ALLOW source → destination` autorise aussi le trafic retour automatiquement (table d'état)
- Default deny : tout ce qui n'est pas explicitement autorisé est bloqué
**Test de validation** : Tu peux expliquer pourquoi `ping 10.0.30.2` depuis 10.0.20.x est bloqué mais `ping 10.0.20.x` depuis 10.0.30.2 fonctionne, avec la même règle de firewall.
### Linux CLI — 🔴 Maîtrise requise
**Pourquoi** : Le Docker Host (Ubuntu) est ton plan de gestion. Tu dois être à l'aise en CLI pour déboguer, éditer des configs, lire des logs.

**Commandes essentielles** :
```bash
# Navigation et fichiers
ls -la, cd, cat, tail -f, grep, find, chmod, chown, ln -s

# Processus et services
ps aux | grep <nom>, systemctl status|start|stop|restart, journalctl -u <service> -f

# Réseau
ip a, ip route, ss -tlnp, curl -v, nc -zv <host> <port>, ping

# Docker
docker compose ps, docker compose logs --tail=50 <service>, docker exec -it <container> bash
docker stats, docker system df, docker compose down/up/restart

# Ansible
ansible -m ping all, ansible-playbook -v playbook.yml --check

# Utilitaires
htop, df -h, du -sh *, free -h, uptime, screen ou tmux
```
### [[Docker]] & Docker Compose — 🔴 Maîtrise requise

**Pourquoi** : Toute la stack tourne dans Docker. Déboguer un conteneur qui crash, modifier une config, gérer les volumes — tout passe par Docker.

**Commandes essentielles** :
```bash
# État des conteneurs
docker compose ps
docker compose logs --tail=50 --follow <service>
docker stats  # consommation CPU/RAM en temps réel

# Gestion du cycle de vie
docker compose up -d           # démarrer en arrière-plan
docker compose restart <service>
docker compose down            # arrêter (garde les volumes)
docker compose down -v         # arrêter ET supprimer les volumes (DANGER)

# Debug dans un conteneur
docker exec -it <container_name> bash
docker exec -it <container_name> sh  # si bash absent

# Volumes et réseaux
docker volume ls
docker volume inspect <volume_name>
docker network ls
docker network inspect <network_name>

# Nettoyage
docker system prune            # supprime les ressources non utilisées
docker system df               # espace disque utilisé par Docker
```

**Points critiques pour ce projet** :
- `OpenSearch heap cap` : TOUJOURS définir `-Xms2g -Xmx2g` dans les env vars
- `volumes:` dans docker-compose = persistance. Sans volume, les données sont perdues au `down`
- `restart: unless-stopped` : les conteneurs redémarrent automatiquement après un reboot du host
- `depends_on:` : ne garantit pas que le service est "prêt", seulement qu'il est "démarré"
- `healthcheck:` : le vrai indicateur de disponibilité d'un service
### DNS
#### Délégation de zone — 🟠 Pratique
**Pourquoi** : [[Cloudflare]] est l'autorité DNS de `charif-labs.tech`. 
* Comprendre la chaîne NS → A/CNAME est nécessaire pour déboguer pourquoi `auth.charif-labs.tech` ne résout pas.

**Ce que tu dois comprendre** :
- Enregistrement NS : délègue l'autorité d'une zone à un serveur DNS
- Propagation : peut prendre jusqu'à 48h (mais Cloudflare est < 5 min)
- CNAME vs A : CNAME pointe vers un autre nom, A pointe vers une IP
- `nslookup -type=NS charif-labs.tech` : vérifie qui est autoritaire
- `dig @1.1.1.1 auth.charif-labs.tech` : résolution forcée via Cloudflare
#### Tunnels 

#### Subdomains

### [[Terraform]] (provider Cloudflare) — 🟠 Pratique

**Pourquoi** : Tout le DNS et les tunnels Cloudflare sont gérés par Terraform. 
- Modifier à la main dans la console Cloudflare = état divergent = problèmes.

**Ce que tu dois maîtriser** :
- `terraform init` : télécharge les providers
- `terraform plan` : montre ce qui va changer AVANT d'appliquer
- `terraform apply` : applique les changements
- `terraform state list` : liste les ressources trackées
- `terraform import` : intégrer une ressource existante dans le state
- Variables sensibles : ne JAMAIS mettre dans le code, utiliser des variables d'environnement
- `.terraform.lock.hcl` : ne pas modifier manuellement

---
## Phase 1 — Identité & ZTNA

### [[OAuth 2.0]] & [[OpenID Connect (OIDC)]] — 🔴 Maîtrise requise

**Pourquoi** : Authelia parle OIDC à Authentik. Grafana parle OIDC à Authentik. Entra ID parle OIDC à Authentik (source fédérée). Sans comprendre ce flux, tu configures à l'aveugle et tu ne peux pas déboguer.

**OAuth 2.0 — Les 4 rôles** :
- **Resource Owner** : l'utilisateur humain
- **Client** : l'application qui veut accéder (Authelia, Grafana)
- **Authorization Server** : celui qui délivre les tokens (Authentik)
- **Resource Server** : ce qu'on veut accéder (Wazuh, TRMM)

**Authorization Code Flow (le plus courant)** :
```
Client → Authorization Server : "Je veux accéder pour l'utilisateur X"
Authorization Server → User : "Login + consent"
User → Authorization Server : Credentials OK
Authorization Server → Client : code (éphémère, 30s)
Client → Authorization Server : code + client_secret → access_token + refresh_token + ID_token
Client → Resource Server : "Authorization: Bearer <access_token>"
```

**OIDC = OAuth 2.0 + couche identité** :
- L'ID Token est un JWT qui répond à "qui est cet utilisateur ?"
- Claims standard : `sub` (identifiant unique), `email`, `name`, `groups`
- Scopes OIDC : `openid` (obligatoire), `profile`, `email`, `groups`
- Discovery endpoint : `https://auth.charif-labs.tech/application/o/<app>/.well-known/openid-configuration`

**Test de validation** : Tu peux expliquer ce qui se passe quand tu cliques sur "Login" dans Grafana jusqu'à ce que tu sois connecté, avec le nom de chaque étape.
### [[SAML 2.0]] — 🔴 Maîtrise requise

**Pourquoi** : La fédération Authentik → Entra ID utilise SAML. C'est aussi ce que Google Workspace utilise en production. Sans comprendre le flux assertion, tu ne peux pas déboguer avec SAML Tracer.

**Les rôles SAML** :
- **IdP (Identity Provider)** : Authentik — émet les assertions, connaît les utilisateurs
- **SP (Service Provider)** : Entra ID — consomme les assertions, donne l'accès

**Le flux SP-initiated** :
```
User → SP (Entra) : "Je veux accéder"
SP → User : Redirect vers IdP avec SAMLRequest
User → IdP (Authentik) : SAMLRequest
IdP → User : Login
User → IdP : Credentials OK
IdP → User : HTML form avec SAMLResponse (base64)
User → SP : POST SAMLResponse (HTTP-POST binding)
SP → vérifie la signature → crée une session → accorde l'accès
```

**Éléments critiques d'une assertion SAML** :
- `Issuer` : qui a émis l'assertion (Entity ID de l'IdP)
- `NameID` : identifiant unique de l'utilisateur (email ou username)
- `Conditions` : validité temporelle (NotBefore, NotOnOrAfter)
- `AttributeStatement` : claims mappés (email, groups)
- `Signature` : signée avec la clé privée de l'IdP, vérifiée avec la clé publique

**Outil obligatoire** : [SAML Tracer](https://addons.mozilla.org/firefox/addon/saml-tracer/) — installe-le avant de commencer la configuration SAML.
### [[JWT — JSON Web Tokens]] — 🟠 Pratique

**Pourquoi** : Les tokens OIDC (ID Token, Access Token) sont des JWTs. Authelia crée des sessions JWT. Déboguer un token expiré ou mal signé nécessite de savoir lire un JWT.

**Structure** : `<Header>.<Payload>.<Signature>` (base64url encodé)

```json
// Header
{ "alg": "RS256", "typ": "JWT" }

// Payload (claims)
{
  "sub": "user123",           // Subject (identifiant unique)
  "iss": "https://auth.charif-labs.tech/", // Issuer
  "aud": "grafana",           // Audience (qui peut utiliser ce token)
  "exp": 1700000000,          // Expiration (Unix timestamp)
  "iat": 1699996400,          // Issued At
  "email": "user@example.com",
  "groups": ["admins", "operators"]
}
```

**Outil** : https://jwt.io — colle un token, inspecte tous les champs.
### [[Authentik]] — Configuration avancée — 🟠 Pratique

**Pourquoi** : Authentik est l'IdP de tout le projet. Une mauvaise configuration des flows bloque l'accès à tout.

**Concepts clés** :
- **Flow** : séquence de stages (ex: Authentication Flow = Identifier → Password → MFA)
- **Stage** : une étape dans un flow (ex: `PasswordStage`, `AuthenticatorTOTPStage`)
- **Binding** : associe un stage à un flow avec un ordre d'exécution
- **Provider** : une application (OIDC Provider, SAML Provider)
- **Application** : ce qu'un utilisateur voit dans le portail (lié à un Provider)
- **Outpost** : composant proxy embarqué (utilisé pour le Embedded Outpost)

**Ordre de configuration** :
1. Créer les OUs (groupes hiérarchiques)
2. Créer les utilisateurs + assigner aux groupes
3. Créer le Provider (OIDC ou SAML)
4. Créer l'Application (liée au Provider)
5. Tester le flux SSO
6. Configurer les flows MFA
### Cloudflare Tunnels & Zero Trust
**cloudflared, tunnel UUID, ingress rules, ZTNA policies**

---
## Phase 2 — Stack de management
### [[Ansible]] — [[Windows (WinRM)]] — 🔴 Maîtrise requise

**Pourquoi** : Ansible configure tout sur les VMs Windows. Le CIS hardening, le déploiement des agents, la configuration Sysmon — tout passe par Ansible + WinRM.

**Prérequis WinRM** :
```powershell
# Sur chaque VM Windows (PowerShell en admin) :
winrm quickconfig -force
winrm set winrm/config/service/auth '@{Basic="true"}'
winrm set winrm/config/service '@{AllowUnencrypted="false"}'

# Créer un certificat auto-signé pour HTTPS
$cert = New-SelfSignedCertificate -DnsName "lab-basic-01" -CertStoreLocation "cert:\LocalMachine\My"
New-Item -Path WSMan:\localhost\Listener -Transport HTTPS -Address * -CertificateThumbprint $cert.Thumbprint -Force

# Ouvrir le firewall
New-NetFirewallRule -Name "WinRM-HTTPS" -DisplayName "WinRM HTTPS" -Protocol TCP -LocalPort 5986 -Action Allow
```

**Configuration Ansible** :
```ini
# inventory.ini
[windows]
lab-basic-01 ansible_host=10.0.20.101
lab-admin-01 ansible_host=10.0.20.102

[windows:vars]
ansible_user=Administrator
ansible_password={{ vault_winrm_password }}
ansible_connection=winrm
ansible_winrm_transport=ntlm
ansible_winrm_port=5986
ansible_winrm_scheme=https
ansible_winrm_server_cert_validation=ignore
```

**Test de base** :
```bash
ansible windows -m win_ping  # doit retourner "pong"
```
### [[GCPW (Google Credential Provider)]]
**Déploiement, configuration registry, intégration Google Cloud Identity**
### [[Entra ID]] 

### Backup strategy — [[Backblaze B2]] — 🟠 Pratique

**Pourquoi** : Le backup est un KPI critique. Un backup non testé n'est pas un backup.

**Commandes essentielles** :
```bash
# Installation rclone (outil de sync vers B2)
curl https://rclone.org/install.sh | sudo bash
rclone config  # configurer le bucket B2

# Backup PostgreSQL (Authentik, TRMM)
docker exec postgresql pg_dump -U authentik authentik | gzip | gpg --symmetric > /backup/authentik-$(date +%Y%m%d).sql.gz.gpg

# Upload vers B2
rclone copy /backup/ b2:sovereign-backups/$(hostname)/

# Vérification
rclone ls b2:sovereign-backups/  # lister les fichiers dans le bucket
```

---
## Phase 3 — Endpoints
### Wazuh (architecture & règles)
**Manager/Agent/Dashboard, règles custom XML, décodeurs, SCA policies**
### Sysmon — Event IDs EDR — 🔴 Maîtrise requise

**Pourquoi** : Sysmon est le capteur EDR. Sans comprendre les Event IDs, les alertes Wazuh n'ont aucun sens.

**Event IDs critiques** :

| EID | Nom | Ce qu'il capture | Cas d'usage sécurité |
|---|---|---|---|
| 1 | Process Create | Processus lancé : PID, PPID, commandline, hash | Détection de `cmd.exe` lancé par un navigateur |
| 3 | Network Connection | Connexion réseau : processus, IP src/dst, port | C2 communication, exfiltration |
| 7 | Image Load | DLL chargée dans un processus | DLL injection, persistence |
| 10 | Process Access | Accès à la mémoire d'un autre processus | Dump LSASS (Mimikatz) |
| 11 | File Create | Fichier créé ou modifié | Ransomware, dropper |
| 13 | Registry Value Set | Clé de registre modifiée | Persistence (Run key) |
| 22 | DNS Query | Requête DNS par processus | C2 over DNS |

**Config recommandée** : [sysmon-modular (Olaf Hartong)](https://github.com/olafhartong/sysmon-modular) — configuration XML exhaustive et maintenue.
### APIs Cloud (Graph + Admin SDK)
**MS Graph API auth (OAuth2), Google Admin SDK, appels REST, pagination**
### MITRE ATT&CK — 🟠 Pratique

**Pourquoi** : Chaque alerte Wazuh peut être mappée à une technique ATT&CK. C'est le langage commun de la détection.

**Techniques les plus importantes pour ce projet** :

| Technique                         | ID        | Event Sysmon   | Exemple                  |
| --------------------------------- | --------- | -------------- | ------------------------ |
| OS Credential Dumping             | T1003     | EID 10 (LSASS) | Mimikatz                 |
| Process Injection                 | T1055     | EID 7, EID 8   | Shellcode injection      |
| Scheduled Task                    | T1053.005 | EID 1, EID 11  | Persistence via schtasks |
| Registry Run Keys                 | T1547.001 | EID 13         | HKCU\Run                 |
| Command and Scripting Interpreter | T1059     | EID 1          | PowerShell, cmd.exe      |
| Lateral Movement via SMB          | T1021.002 | EID 3          | PsExec, SMB shares       |
### [[Tactical RMM]]
## Phase 4
### n8n — Webhooks + HMAC — 🔴 Maîtrise requise

**Pourquoi** : n8n est le SOAR. Un webhook non sécurisé = n'importe qui peut déclencher des actions sur tes endpoints.

**Validation HMAC-SHA256** :
```javascript
// Dans un Function node n8n :
const crypto = require('crypto');
const secret = $env.WEBHOOK_HMAC_SECRET;  // Storé dans les variables d'env n8n
const payload = JSON.stringify($input.item.json);
const signature = $input.item.headers['x-webhook-signature'];

const expectedSig = 'sha256=' + crypto.createHmac('sha256', secret).update(payload).digest('hex');

if (signature !== expectedSig) {
  throw new Error('Invalid HMAC signature — request rejected');
}
return $input.item;
```
### Tactical RMM — API & scripts
**API token, scripts PowerShell via TRMM, isolation réseau, LAPS**

---
## Phase 5 — Observabilité

### PromQL — 🟠 Pratique

**Pourquoi** : PromQL est le langage des règles Alertmanager et des dashboards Grafana. Sans PromQL, tu ne peux pas écrire tes propres alertes.

**Requêtes essentielles** :
```promql
# CPU usage par instance (%)
100 - (avg by(instance) (rate(windows_cpu_time_total{mode="idle"}[5m])) * 100)

# RAM utilisée (%)
100 * (1 - (windows_os_physical_memory_free_bytes / windows_cs_physical_memory_bytes))

# Disk usage (%)
100 * (1 - (windows_logical_disk_free_bytes{volume="C:"} / windows_logical_disk_size_bytes{volume="C:"}))

# Service down (UP/DOWN)
up{job="windows-exporter"}

# Taux de requêtes HTTP (erreurs 5xx)
rate(http_requests_total{status=~"5.."}[5m])
```

**Fonctions clés** :
- `rate(counter[5m])` : taux de variation d'un counter sur 5 minutes
- `avg by(label)` : moyenne par valeur d'un label
- `sum without(label)` : somme en excluant un label
- `>` `<` `==` : opérateurs de comparaison pour les alertes

---

## Ressources — Références rapides

| Sujet | Ressource | URL |
|---|---|---|
| Authentik docs | Documentation officielle | https://docs.goauthentik.io |
| OAuth 2.0 interactif | oauth.com | https://www.oauth.com |
| SAML Tracer | Extension Firefox | https://addons.mozilla.org/firefox/addon/saml-tracer/ |
| JWT debugger | jwt.io | https://jwt.io |
| MITRE ATT&CK | Framework complet | https://attack.mitre.org |
| Sysmon config | sysmon-modular | https://github.com/olafhartong/sysmon-modular |
| PromQL cheatsheet | promlabs | https://promlabs.com/promql-cheat-sheet |
| Wazuh docs | Documentation officielle | https://documentation.wazuh.com |
| Tactical RMM docs | Documentation officielle | https://docs.tacticalrmm.com |
| CIS Benchmarks | cisecurity.org | https://www.cisecurity.org/benchmark |
| NIST SP 800-207 (ZTA) | nist.gov | https://csrc.nist.gov/publications/detail/sp/800-207/final |
| NIST SP 800-61r2 (IR) | nist.gov | https://www.nist.gov/publications/computer-security-incident-handling-guide |
| Ansible Windows modules | docs.ansible.com | https://docs.ansible.com/ansible/latest/collections/ansible/windows/ |
| WireGuard whitepaper | wireguard.com | https://www.wireguard.com/papers/wireguard.pdf |
| Cloudflare Tunnel docs | developers.cloudflare.com | https://developers.cloudflare.com/cloudflare-one/connections/connect-networks |
