---

tags: [concept, sécurité, zero-trust]

liens: [Architecture]

---
## Définition

Modèle de sécurité qui abandonne le paradigme "faire confiance au réseau interne". Chaque accès est vérifié explicitement, peu importe d'où il vient.

## Les 3 principes fondamentaux

| Principe | Signification | Dans ce projet |
|---|---|---|
| Never trust, always verify | Même à l'intérieur du réseau, chaque accès est authentifié | Cloudflare Access enforced devant chaque service |
| Least privilege | Accès minimum nécessaire, JIT pour le reste | LAPS, JIT elevation via n8n |
| Assume breach | Concevoir comme si l'attaquant est déjà dedans | VLAN isolation, Wazuh EDR, Sysmon |

## Les 5 piliers NIST ZTA (SP 800-207)

1. **Identity** — [[../20 - Outils/Authentik]] + [[Entra ID]] + Cloudflare Access
2. **Device** — [[Action1]] + [[Wazuh]] SCA + Sysmon
3. **Network** — [[../../OPNsense]] VLANs + [[Zero Trust Tunnel]]
4. **Application** — Cloudflare Access (edge) + Authentik Outpost (interne)
5. **Data** — LUKS chiffrement volume, Ansible Vault, backups chiffrés

## Zero Trust ≠ VPN

Un VPN donne accès à tout le réseau interne après authentification. Zero Trust donne accès à **un seul service**, après vérification de l'identité **et** du contexte (device health, heure, localisation).

## Implémentation dans ce projet
![Implementation](Canvas/Implementation.canvas)

## Référence

- NIST SP 800-207 — Zero Trust Architecture (lire résumé exécutif + sections 2-3)
- Google BeyondCorp Papers — le modèle qui a inspiré le zero trust moderne
