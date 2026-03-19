---

tags: [concept, identité, protocole, saml, federation]

liens: [Authentik, Entra ID]

---
## Définition
SAML (Security Assertion Markup Language) est un protocole XML d'échange d'informations d'authentification entre un **Identity Provider** et un **Service Provider**.
## IdP vs SP

| Rôle | Définition | Dans ce projet |
|---|---|---|
| **IdP** (Identity Provider) | Connaît l'utilisateur, émet les assertions | [[../20 - Outils/Authentik]] |
| **SP** (Service Provider) | Consomme les assertions, accorde l'accès | [[Entra ID]] Free |
## SAML Assertion
L'assertion est un document XML signé numériquement par l'IdP, qui contient :
- L'identité de l'utilisateur (NameID, attributs)
- Les conditions de validité (dates, audience)
- La signature cryptographique (clé privée IdP)

Le SP vérifie la signature avec la clé publique de l'IdP (issue du metadata XML).
## SP-initiated SSO Flow
```
1. Utilisateur tente d'accéder à Entra ID
2. Entra ID (SP) redirige → Authentik (IdP) avec une SAML AuthnRequest
3. Authentik authentifie l'utilisateur (si pas déjà de session)
4. Authentik génère une SAML Response (XML signé)
5. HTTP POST de l'assertion vers l'ACS URL d'Entra ID
6. Entra ID vérifie la signature, crée la session
7. Accès accordé au service Microsoft
```
## Metadata XML — le contrat IdP ↔ SP
Chaque partie expose un fichier XML de metadata contenant :
- **Entity ID** : identifiant unique de l'entité
- **ACS URL** (Assertion Consumer Service) : où envoyer les assertions (côté SP)
- **Certificat de signature** : clé publique pour vérifier les assertions (côté IdP)
Pour que la fédération fonctionne, chaque partie doit avoir importé le metadata de l'autre.
## Attribute mapping
L'IdP envoie des attributs dans l'assertion. Le SP doit les mapper sur ses propres champs :

| Authentik claim | Entra ID attribute |
|---|---|
| `email` | `user.mail` |
| `preferred_username` | `user.userprincipalname` |
| `groups` | rôles ou groupes Entra |
## Debugger la fédération
**SAML Tracer** (extension Firefox/Chrome) — capture les échanges SAML en temps réel. 
Indispensable pour identifier les erreurs de configuration (mauvais Entity ID, ACS URL incorrecte, assertion expirée, signature invalide).
## Dans ce projet
- Authentik configure une **SAML Provider** pointant vers Entra ID (ACS URL + Entity ID)
- Entra ID configure une **Enterprise Application** avec le metadata d'Authentik
- Résultat : login Windows avec credentials Authentik, via Entra Join
Voir [[Phase 1 - Fédération OIDC Entra]] pour les étapes de configuration.
