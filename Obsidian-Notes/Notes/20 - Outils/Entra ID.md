---

tags: [outil, identité, microsoft, sp]

concepts: [SAML Federation, OIDC Flow]

---
## Rôle dans ce projet

Service Provider (SP) côté Windows. Reçoit les assertions SAML depuis [Keycloak](Keycloak) (IdP).  
Permet : Entra Join des VMs Windows sans Intune ni MDM enrollment.

**Entra ID Free ≠ Intune.** Entra Join ne nécessite pas de licence Intune. Il lie simplement le device au tenant Entra pour l'authentification.

## Tenant

Domaine : `ms.charif-labs.tech`  
Vérification : TXT record `MS=ms76330167` (en place via [[../../Terraform]])  
URL portail : `https://entra.microsoft.com`

## Entra Join — login Windows sans AD

```powershell
# Sur la VM Windows, en admin
dsregcmd /join

# Vérifier le statut
dsregcmd /status
# AzureAdJoined : YES  ← ce qu'on veut voir
```

Cela lie le device au tenant Entra. Le login Windows utilise ensuite les credentials fédérés depuis keycloak via SAML.

## Enterprise Application (fédération SAML)

Pour que Entra accepte les assertions SAML d'keycloak :

1. Entra Admin → Enterprise Applications → New Application → Create your own
2. Configure Single Sign-On → SAML
3. Renseigner :
   - **Entity ID** : correspond à l'issuer configuré dans keycloak
   - **Reply URL (ACS)** : URL vers laquelle keycloak envoie l'assertion
   - **Sign-on URL** : URL de login Entra
1. Télécharger le Certificate (Base64) → importer dans keycloak
2. Copier les metadata → importer dans keycloak Provider SAML

## App Registration (fédération OIDC)

Pour que keycloak puisse fédérer les identités Microsoft :

1. Entra Admin → App Registrations → New Registration
2. Nom : `keycloak-federation`
3. Redirect URI : `https://auth.charif-labs.tech/source/oauth/callback/entra/`
4. Générer un Client Secret (noté immédiatement — ne s'affiche qu'une fois)
5. Configurer dans keycloak : Sources → OAuth/OIDC → Microsoft/Entra

## Compte break-glass

Un compte natif Entra (pas fédéré) à utiliser si keycloak est down.  
Mot de passe imprimé, sous pli scellé.  
Format : `breakglass@<tenant>.onmicrosoft.com`
