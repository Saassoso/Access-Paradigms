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
### Docker & Docker Compose
Volumes, networks bridge/host, .env, docker compose ps/logs/restart, cap mémoire
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

### OAuth 2.0 & OpenID Connect (OIDC) — 🔴 Maîtrise requise

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
### SAML 2.0 — 🔴 Maîtrise requise

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
### Authentik (configuration)
**Providers OIDC upstream, Outposts, flows d’authentification, applications**
### Cloudflare Tunnels & Zero Trust
**cloudflared, tunnel UUID, ingress rules, ZTNA policies**
## Phase 2
### Ansible — Windows (WinRM)
**Inventaire, playbooks, roles, vault, connexion WinRM, modules win_***
### GCPW (Google Credential Provider)
**Déploiement, configuration registry, intégration Google Cloud Identity**
## Phase 3
### Wazuh (architecture & règles)
**Manager/Agent/Dashboard, règles custom XML, décodeurs, SCA policies**
### Sysmon — Event IDs EDR
**Events 1/3/10, config sysmon-modular (Olaf Hartong), lsass.exe, MITRE ATT&CK**
### APIs Cloud (Graph + Admin SDK)
**MS Graph API auth (OAuth2), Google Admin SDK, appels REST, pagination**
## Phase 4
### n8n — Webhooks HMAC
**Nodes HTTP/webhook, validation HMAC-SHA256, JSON parsing, appels API REST**
### Tactical RMM — API & scripts
**API token, scripts PowerShell via TRMM, isolation réseau, LAPS**