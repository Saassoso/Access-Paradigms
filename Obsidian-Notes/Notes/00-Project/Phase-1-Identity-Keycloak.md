---
tags: [phase, guide, keycloak, oidc, ztna]
phase: 1
statut: 🔄 En cours
---
# Phase 1 — Identity Broker & ZTNA

> **Objectif :** Un seul login qui ouvre tous les outils internes. MFA activé. Cloudflare Access protège tout.  
> **IdP :** Keycloak (déjà déployé sur `auth.charif-labs.tech`)

---

## Prérequis Phase 1

- [x] Phase 0 validée (VLAN, DNS, Cloudflare Tunnel actif)
- [x] Keycloak déployé sur `auth.charif-labs.tech`
- [x] Realm `charif-labs` créé dans Keycloak
- [x] Google Cloud Identity tenant validé (`google.charif-labs.tech`)

---

## 1.1 — Finaliser la Configuration Keycloak

### Groupes à créer / vérifier

Keycloak Admin → `charif-labs` realm → Groups :

| Groupe | Utilisateurs | VLAN | Profil cloud |
|---|---|---|---|
| `bureau-1` | Basic users | 20 | Google-Only |
| `bureau-2-it` | Admins IT | 20 | Microsoft-Only |
| `bureau-2-dev` | Développeurs | 20 | Microsoft-Only |
| `bureau-2-compta` | Comptabilité | 20 | Microsoft-Only |
![](../99%20-%20Attachment/images/Phase-1-Identity-Keycloak.png)
### Rôles associés aux groupes

Keycloak → Realm roles → créer puis assigner via Groups → Role Mappings :

```
basic-user  → bureau-1
```
![](../99%20-%20Attachment/images/Phase-1-Identity-Keycloak-1.png)
```
it-admin    → bureau-2-it
```
![](../99%20-%20Attachment/images/Phase-1-Identity-Keycloak-2.png)
```
dev         → bureau-2-dev
```
![](../99%20-%20Attachment/images/Phase-1-Identity-Keycloak-3.png)
```
compta      → bureau-2-compta
```
![](../99%20-%20Attachment/images/Phase-1-Identity-Keycloak-4.png)
### MFA TOTP obligatoire pour les admins

Keycloak → Authentication → Required Actions → Configure OTP → Default Action : ON
![](../99%20-%20Attachment/images/Phase-1-Identity-Keycloak-5.png)

- Pour forcer uniquement sur certains groupes :
```
Keycloak → Authentication → Policies → Conditional OTP Policy
  Condition user role : it-admin, dev
  OTP Control User : Force
```

---
## 1.2 — Cloudflare Access sur tous les services

### Créer le Client OIDC Keycloak pour Cloudflare

Keycloak → Clients → Create client

```
Client ID   : cloudflare-access
Client type : OpenID Connect
Name : Cloudflare Access
Valid redirect URIs : https://charif-labs.cloudflareaccess.com/cdn-cgi/access/callback

Client auth : ON (confidential)
```

Onglet **Client scopes** → `cloudflare-access-dedicated` → Add mapper → Groups :
![](../99%20-%20Attachment/images/Phase-1-Identity-Keycloak-6.png)
```
Name            : groups
Token Claim Name: groups
Add to ID token : ON
Add to userinfo : ON
```

Copier le `Client Secret` depuis l'onglet **Credentials**.
![](../99%20-%20Attachment/images/Phase-1-Identity-Keycloak-7.png)
### Dans Cloudflare Zero Trust Dashboard

Access in terraform 
- add zero trust to api key {Account > Zero Trust > Edit}
- [[../Notes - to Remember]]
```
Name         : Keycloak
Client ID    : cloudflare-access
Client Secret: [depuis Keycloak → Clients → cloudflare-access → Credentials]
Auth URL     : https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/auth
Token URL    : https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/token
JWKS URL     : https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/certs
```
### Policies Access par service
![](../99%20-%20Attachment/images/Phase-1-Identity-Keycloak-8.png)

| Service         | Domaine                    | Groupe requis |
| --------------- | -------------------------- | ------------- |
| Keycloak        | auth.charif-labs.tech      | Any user      |
| Wazuh           | wazuh.charif-labs.tech     | bureau-2-it   |
| n8n             | n8n.charif-labs.tech       | bureau-2-it   |
| Portainer       | portainer.charif-labs.tech | bureau-2-it   |
| Keyacloak Admin |                            |               |

---
## 1.3 — Portainer
### Déployer Portainer
```yaml
# docker/portainer/docker-compose.yml
services:
  portainer:
    image: portainer/portainer-ce:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    networks:
      - sovereign_net
    restart: unless-stopped
volumes:
  portainer_data:
networks:
  sovereign_net:
    external: true
```
### Client Keycloak pour Portainer
Keycloak → Clients → Create client

```
Client ID  : portainer-sso
Client type: OpenID Connect
Client auth: ON
Valid redirect URIs: https://portainer.charif-labs.tech/*
```
### Configurer OIDC dans Portainer
Portainer → Settings → Authentication → OAuth
```
Client ID       : portainer-sso
Client Secret   : [Keycloak → Clients → portainer-sso → Credentials]
Authorization URL: https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/auth
Access Token URL : https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/token
Resource URL     : https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/userinfo
Redirect URL     : https://portainer.charif-labs.tech/
Logout URL       : https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/logout?client_id=portainer-sso&post_logout_redirect_uri=https://mgmt.charif-labs.tech
Scopes           : openid email profile groups
```
![](../99%20-%20Attachment/images/Phase-1-Identity-Keycloak-9.png)
### Cloudflare Tunnel → Portainer

```hcl
# ingress.tf
ingress_rule {
  hostname = "mgmt.charif-labs.tech"
  service  = "http://portainer:9000"
}
```

```bash
terraform apply
curl -I https://mgmt.charif-labs.tech
# Attendu : 302 → Cloudflare Access
```
![](../99%20-%20Attachment/images/Phase-1-Identity-Keycloak-10.png)
---

## 1.4 — Identity Provider Microsoft (bouton "Se connecter avec Microsoft")

> Permettre aux utilisateurs (ex: Bureau-2) de s'authentifier sur Keycloak en utilisant directement leur compte Microsoft Entra ID (ex: `utilisateur@ms.charif-labs.tech`). Keycloak délègue l'authentification à Microsoft via le protocole OpenID Connect (OIDC).
### App Registration Entra ID
Permet de déclarer Keycloak comme une application autorisée à demander des authentifications à Microsoft.

1. Entra Admin → App Registrations → New Registration
2. Nom : `keycloak-idp-microsoft`
3. Redirect URI (Web) : `https://auth.charif-labs.tech/realms/charif-labs/broker/microsoft/endpoint`
4. Copier Application (client) ID + générer Client Secret

### Identity Provider dans Keycloak
Indique à Keycloak comment contacter l'application Entra ID fraîchement créée.

1. Keycloak → `charif-labs` realm → Identity Providers → Add provider → Microsoft
```
Alias        : microsoft
Display Name : Se connecter avec Microsoft
Client ID    : [Application ID Entra]
Client Secret: [Secret Value Entra]
```
### SAML Client 
- Application qui demandee l'ahtentification de l'utilisateur 
	- urn:federation:MicrosoftOnline
```
Client ID                  :   urn:federation:MicrosoftOnline
Name                       :   Microsoft Entra ID Client
Valid redirect URIs        :   https://login.microsoftonline.com/login.srf
Master SAML Processing URL :   https://login.microsoftonline.com/d56b5879-c621-4f54-baf4-47dc342da17a/saml2

Name Id Format             :   email
```
### Paramètres Avancés et Synchronisation 
Pour éviter les boucles de connexion et s'assurer que les utilisateurs sont importés proprement 
#### Enta ID :
Domain is Federated, not PHS.
- Type: Custom — Status: Verified — Federated: **Yes** — Primary domain: Yes
It means Entra has been told _"for users @ms.charif-labs.tech, don't authenticate them yourself — redirect to an external IdP (Keycloak SAML)"_. 

so we go to entra federated interface :
``` powershell
Install-Module Microsoft.Graph -Force -AllowClobber
Connect-MgGraph -Scopes "Domain.ReadWrite.All"
Update-MgDomain -DomainId "ms.charif-labs.tech" -AuthenticationType "Managed"
```

#### Keycloak :
Descendez dans les paramètres de l'Identity Provider :

- **Trust Email :** `On` _(Évite de demander à l'utilisateur de valider son email après l'import)._
- **Sync Mode :** `Import` _(Crée une copie locale de l'utilisateur dans la base Keycloak à la première connexion)._
- **First login flow :** `first broker login`
- **Prompt :** `login` _(Force Microsoft à toujours afficher l'écran de connexion au lieu d'utiliser un token silencieux en cache. C'est une sécurité supplémentaire contre les boucles de redirection)._
	- Tells Entra what to show. Options: blank (Entra decides), `login` (always show Entra login screen), `none` (silent SSO only), `select_account`.
### Test
Bouton "Se connecter avec Microsoft" visible sur `auth.charif-labs.tech` 
![](../99%20-%20Attachment/images/Phase-1-Identity-Keycloak-11.png)
→ login `x@ms.charif-labs.tech` 
![](../99%20-%20Attachment/images/Phase-1-Identity-Keycloak-12.png)

→ session Keycloak active ✅
![](../99%20-%20Attachment/images/Phase-1-Identity-Keycloak-13.png)


---

## 1.5 — Identity Provider Google (bouton "Se connecter avec Google")

 >Permettre aux utilisateurs de s'authentifier sur Keycloak en utilisant leur compte Google Workspace. L'authentification est déléguée à Google via le protocole OpenID Connect (OAuth 2.0).

#### Étape 1 : Création des identifiants dans Google Cloud Console
Même si vous utilisez Google Workspace, la création des identifiants se fait obligatoirement sur l'interface développeur de Google.
1. Connectez-vous avec votre compte administrateur Workspace sur **console.cloud.google.com**.
2. En haut à gauche, cliquez sur le menu déroulant des projets et sélectionnez **Nouveau projet** (New Project). Nommez-le `Keycloak-SSO` et créez-le.
3. Allez dans le menu principal → **API et services** (APIs & Services) → **Écran de consentement OAuth**.
4. **Audience :** Sélectionnez **Interne** (Internal). Cela sécurise l'accès à votre Workspace et ne nécessite aucune validation par Google.
5. **App Information :** Saisissez le nom `Charif Labs SSO` ou `Keycloak` (le nom visible par les utilisateurs) et sélectionnez votre adresse email Workspace pour le support.
6. **Contact Information :** Saisissez une adresse email pour être contacté en cas de changements sur le projet.
7. **Finish :** Acceptez les étapes suivantes et sauvegardez pour terminer la création de l'écran.
#### Étape 2 : Création des identifiants (Client ID et Secret)
1. Google Cloud Console → APIs & Credentials → Client → Create OAuth client ID → Create
   - Type : Web application
   - **Nom :** `keycloak-idp-google`
   - Redirect URI : `https://auth.charif-labs.tech/realms/charif-labs/broker/google/endpoint`
2. Copier Client ID + Secret
#### Étape 3 : Configuration dans Keycloak
Keycloak → Identity Providers → Add provider → Google

```
Alias        : google
Display Name : Se connecter avec Google
Client ID    : [depuis Google Cloud Console]
Client Secret: [depuis Google Cloud Console]
```
#### Étape 4 : Paramètres Avancés et Synchronisation (Important)

Pour assurer une création propre des comptes et forcer la sélection du compte Google à chaque connexion :
- **Trust Email :** `On` _(Google a déjà vérifié l'email, cela évite l'étape de validation)._
- **Sync Mode :** `Import` _(Crée une copie locale de l'utilisateur dans la base Keycloak)._
- **First login flow :** `first broker login`
- **Prompt :** `select_account` _(Force Google à afficher l'écran de choix de compte. C'est l'équivalent du `login` chez Microsoft, très utile si les utilisateurs ont un compte personnel et un compte pro sur le même navigateur)._
#### Étape 4 : Test de Validation
1. Déconnectez-vous de Keycloak.
2. Allez sur l'URL de connexion : `https://auth.charif-labs.tech`
3. Vérifiez que le bouton **"Se connecter avec Google"** est visible.
4. Cliquez dessus, choisissez un compte de votre Google Workspace, et validez.
5. La session Keycloak doit s'ouvrir avec succès ! ✅
---

## Endpoints Keycloak de Référence

```
Discovery : https://auth.charif-labs.tech/realms/charif-labs/.well-known/openid-configuration
Auth      : https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/auth
Token     : https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/token
JWKS      : https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/certs
Userinfo  : https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/userinfo
Logout    : https://auth.charif-labs.tech/realms/charif-labs/protocol/openid-connect/logout
Admin UI  : https://auth.charif-labs.tech/admin/charif-labs/console/
```

---

## Configure role based MFA 
![](../99%20-%20Attachment/images/Phase-1-Identity-Keycloak-15.png)


![](../99%20-%20Attachment/images/Phase-1-Identity-Keycloak-14.png)
## Validation Gatekeeper Phase 1

```bash
# Discovery endpoint opérationnel
curl -s https://auth.charif-labs.tech/realms/charif-labs/.well-known/openid-configuration | jq .issuer

# Portainer redirige vers Cloudflare Access
curl -I https://mgmt.charif-labs.tech

# Logs Keycloak — connexions
docker compose logs keycloak-1 | grep "type=LOGIN"
```

| Test                           | Attendu                        | ✅/❌ |
| ------------------------------ | ------------------------------ | --- |
| Portainer sans auth            | 302 → Cloudflare Access        | ✅   |
| Login user-admin-01            | MFA TOTP demandé               | ✅   |
| Login user-basic-01            | Pas de MFA                     | ✅   |
| Accès Wazuh avec user-basic-01 | Refusé par Cloudflare Access   |     |
| Discovery endpoint             | Répond 200 avec issuer correct |     |
