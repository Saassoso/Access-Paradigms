---
tags: [tasks, phase-2, suivi]
statut: en-cours
date-maj: 2026-06-18
---

# 📋 Tâches — Phase 2 : Endpoints Windows

> Phase en cours. Compléter Bureau-2 (Entra) avant Bureau-1 (GCPW).  
> Guide détaillé : [[../00-Project/Phase-2-Endpoint-Windows]]

---

## Bureau-2 — Entra ID (Priorité)

### App Registration & Vault
- [ ] Stocker Client Secret `n8n-iam-sync` dans Vault (pas dans `.env`)
- [ ] Tester création utilisateur Entra via Postman/Graph Explorer (validation manuelle)

### Onboarding Windows
- [ ] Script `bootstrap_entra.ps1` : Entra Join + Action1 agent
- [ ] VM Windows Bureau-2 jointe à Entra ID
  - Validation : `dsregcmd /status` → `AzureAdJoined: YES`
- [ ] **Test login Windows** avec `user-admin-01@ms.charif-labs.tech` ← Test final

---

## Bureau-1 — GCPW (Après Bureau-2 validé)

- [ ] Créer Service Account GCP avec Domain-Wide Delegation
  - Scope : `admin.directory.user` (restreint à OU Bureau-1 uniquement)
- [ ] Télécharger GCPW MSI (domaine = `google.charif-labs.tech`)
- [ ] Script `bootstrap_gcpw.ps1` : installation GCPW silencieuse + Action1 agent
- [ ] **Test login Windows** avec compte Google Workspace

---

## Action1 RMM (Commun aux deux profils)

- [ ] Créer compte Action1 + signer DPA RGPD
- [ ] Créer groupe `Onboarding-Entra` (Bureau-2) + groupe `Onboarding-GCPW` (Bureau-1)
- [ ] Configurer scripts d'onboarding dans chaque groupe :
  - `Apply-RegistryPolicies.ps1`
  - `Set-SecurityPolicies.ps1`
  - `Init-LAPS.ps1`
- [ ] Ajouter attribut `action1_agent_id` dans Keycloak pour chaque utilisateur
- [ ] Test : nouveau PC → rejoint groupe → scripts s'exécutent automatiquement

---

## Ansible Hardening CIS

- [ ] `ansible/inventory.ini` + `win_ping` OK
- [ ] Playbook CIS hardening Windows 11 L1
- [ ] BitLocker activé + clé escrowée dans Entra

---

## ✅ Gatekeeper Phase 2

```
[ ] Bureau-2 : login Windows avec compte Entra → dsregcmd = AzureAdJoined: YES
[ ] Bureau-1 : login Windows avec compte GCPW → session Google
[ ] Action1 : nouveau PC → scripts d'onboarding s'exécutent automatiquement
```

---

## Tâches Complétées (Phase 2)

- [x] Domaine `ms.charif-labs.tech` en mode **Managed** (non Federated) dans Entra
- [x] App Registration `n8n-iam-sync` avec permissions Graph API minimales
  - ✅ `User.ReadWrite.All` + `GroupMember.ReadWrite.All`
  - ✅ PAS de `Directory.ReadWrite.All` (sur-privilège évité)
- [x] Google Cloud Identity tenant validé (`charif-labs.tech`)

---

## Notes & Blocages

_(Ajoute ici les blocages rencontrés et les solutions)_

