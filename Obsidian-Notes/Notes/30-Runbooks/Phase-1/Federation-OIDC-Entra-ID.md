---
tags:
  - tâche
  - phase-1
  - identité
  - federation
  - entra
outils:
  - Entra ID
concepts:
  - OIDC Flow
  - SAML Federation
statut: ❌ à faire
---
## Objectif

Permettre aux utilisateurs de se connecter sur `auth.charif-labs.tech` avec leur compte `@ms.charif-labs.tech`.

Deux configurations à faire en parallèle :
1. **Source OIDC** dans keycloak → fédère les identités Microsoft
2. **Enterprise App SAML** dans Entra → fédère vers Windows

## Étape 1 — App Registration Entra ID (pour Source OIDC keycloak)

1. Entra Admin → App Registrations → New Registration
2. Nom : `keycloak-oidc-source`
3. Redirect URI (Web) : `https://auth.charif-labs.tech/source/oauth/callback/entra/`
4. → Register

5. Copier : Application (client) ID  
6. Certificates & Secrets → New client secret → copier la valeur **immédiatement**

## Étape 2 — Source OIDC dans keycloak

keycloak → Directory → Federation & Social login → Create → Microsoft Entra ID (OIDC)

```
Name : entra-source
Consumer key : <Application ID de l'App Registration>
Consumer secret : <Secret Value>
Tenant ID : <Directory ID du tenant>
```

Test : bouton "Login with Microsoft" apparaît sur auth.charif-labs.tech  
→ Se connecter avec `user@ms.charif-labs.tech`

## Étape 3 — Enterprise App SAML dans Entra (pour login Windows)

1. Entra Admin → Enterprise Applications → New Application → Create your own
2. Nom : `keycloak-saml-sp`
3. Single sign-on → SAML

4. Section "Basic SAML Configuration" :
   - Identifier (Entity ID) : `urn:charif-labs:keycloak`
   - Reply URL (ACS URL) : récupérer depuis keycloak (Provider SAML → Metadata)

5. Télécharger Certificate (Base64)

## Étape 4 — Provider SAML keycloak

keycloak → Applications → Providers → Create → SAML Provider

```
Name : entra-saml-provider
ACS URL : URL de l'Enterprise App Entra
Issuer : urn:charif-labs:keycloak
Signing Certificate : importer le certificate téléchargé depuis Entra
```

## Étape 5 — Test bout en bout

```
1. Naviguer vers https://auth.charif-labs.tech
2. Cliquer "Login with Microsoft"
3. Entrer user@ms.charif-labs.tech + mot de passe
4. Consent → redirection vers keycloak
5. Session active dans keycloak ✅
```

## Debugger avec SAML Tracer

Extension Firefox/Chrome. Capture les assertions SAML en temps réel.  
Éléments à vérifier :
- Entity ID correspondant entre IdP et SP
- ACS URL correcte
- Assertion non expirée (`NotOnOrAfter`)
- Signature valide (certificat importé des deux côtés)