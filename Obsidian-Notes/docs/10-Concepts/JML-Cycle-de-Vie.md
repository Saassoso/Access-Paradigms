---
tags: [concept, iam, cycle-de-vie, provisioning]
liens: [IAM, n8n, Keycloak, PAM]
---

# JML — Cycle de Vie des Identités (Joiner / Mover / Leaver)

## Définition

JML = le modèle qui décrit les 3 événements du cycle de vie d'une identité en entreprise.  
C'est ce que le workflow n8n automatise dans ce projet.

## Les 3 Moments

### 🟢 Joiner — Arrivée d'un utilisateur

**Déclencheur :** Création d'un utilisateur dans Keycloak (admin IT)

**Ce que n8n déclenche automatiquement :**
```
1. Lire le groupe de l'utilisateur (bureau-1 ou bureau-2)
2. Switch :
   ├── bureau-1 → Google Admin SDK → Créer compte Google Workspace
   └── bureau-2 → Graph API → Créer compte Entra ID (PHS)
3. Envoyer les credentials temporaires à l'utilisateur
4. Déclencher le script d'onboarding Action1 (installation agents, policies)
5. Logger l'événement dans Wazuh
```

**SLA cible :** Compte satellite créé en < 2 minutes après création dans Keycloak.

### 🟡 Mover — Changement de rôle/groupe

**Déclencheur :** Modification du groupe dans Keycloak (ex: `bureau-2-dev` → `bureau-2-it`)

**Ce que n8n déclenche :**
```
1. Mettre à jour les groupes dans le satellite (Entra ou Google)
2. Révoquer les anciens accès (ex: retirer du groupe dev dans Entra)
3. Ajouter les nouveaux accès (ex: ajouter au groupe it-admin)
4. Les policies Cloudflare Access se mettent à jour automatiquement (basées sur claims)
5. Logger le changement
```

**Point clé :** Les droits Cloudflare Access sont **basés sur les claims JWT** (groupe Keycloak).  
Un changement de groupe dans Keycloak = nouvelle session = nouveaux droits. Immédiat.

### 🔴 Leaver — Départ d'un utilisateur

**Déclencheur :** Désactivation dans Keycloak (admin IT marque l'utilisateur comme inactif)

**L'ordre d'exécution est critique (ordre corrigé) :**
```
1. Désactiver session Keycloak → invalide TOUS les tokens SSO actifs
2. Désactiver dans satellite (Entra ou Google) → plus de login Windows
3. Changer le mot de passe LAPS → empêche accès local au PC
4. Déconnecter la carte réseau NIC via Action1 → isolation physique du poste
5. Logger l'événement avec timestamp → preuve pour audit
```

**SLA cible : < 90 secondes** de Keycloak à révocation complète.

> ⚠️ Ne jamais commencer par le satellite. Si Keycloak reste actif, l'utilisateur peut toujours obtenir un nouveau token via son session cookie.

## Attribut action1_agent_id

Chaque utilisateur Keycloak doit avoir l'attribut `action1_agent_id` renseigné.  
C'est ce qui permet à n8n de cibler le bon PC lors de l'offboarding (étape NIC disconnect).

## Hostile Leaver — Cas Spécial

Pour un départ conflictuel (licenciement immédiat, suspicion de menace interne) :

```
1. Admin → Keycloak : désactiver l'utilisateur (même procédure, mais en urgence)
2. Wazuh : augmenter le niveau de surveillance du poste les 24h suivantes
3. Exporter les logs Wazuh du poste pour audit
4. Rotation des secrets partagés que l'utilisateur connaissait (Vault → régénérer)
```

## Pourquoi Keycloak comme Source Unique ?

Le principe du projet : **Keycloak = Single Source of Truth**.

L'admin IT ne touche qu'à Keycloak. n8n propage vers Entra et Google automatiquement.  
→ Pas de désynchronisation possible. Pas de compte oublié dans un satellite.

## Liens

- [[IAM]] — Contexte global de la gestion des identités
- [[n8n]] — L'outil qui exécute les workflows JML
- [[Keycloak]] — La source d'événements (webhooks)
- [[PAM]] — LAPS et gestion des comptes privilégiés
- [[00-Project/Phase-4-SOAR-n8n|Phase 4]] — Implémentation détaillée
