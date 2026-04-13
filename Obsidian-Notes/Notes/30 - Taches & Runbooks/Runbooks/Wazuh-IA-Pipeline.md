---
tags: [référence, wazuh, ia, siem, pipeline]
statut: actif
---
# Wazuh + IA — Pipeline d'Analyse Consultatif

> **Principe :** L'IA analyse, l'humain décide. Jamais d'action destructive automatique.

---
## Architecture du Pipeline

```
Wazuh Manager
    │ Alertes niveau ≥ 7 (JSON)
    │
    ├── niveau ≥ 12 → webhook immédiat → n8n /webhook/wazuh-critical
    └── niveau 7-11 → webhook batch (5 min max) → n8n /webhook/wazuh-ai-batch
                                                          │
                                             Pré-filtrage + dédupication
                                                          │
                                               Appel LLM (Claude Haiku)
                                                          │
                                    ┌─────────────────────┼──────────────────────┐
                                    ▼                     ▼                      ▼
                              risk=normal           risk=suspect           risk=critique
                              Log silencieux        Slack + email          Alerte prioritaire
                              OpenSearch            responsable            (action humaine)
```

---
## Fichier de Consignes (wazuh-ai-instructions.txt)

> Versionné dans Git : `/docker/n8n/config/wazuh-ai-instructions.txt`  
> Modifiable par le responsable IT sans toucher au code.

```
Tu es un analyste de sécurité pour l'infrastructure Charif-Labs.
Analyse les alertes Wazuh fournies et classe-les selon leur criticité.

Contexte infrastructure :
- VLAN 20 (10.0.20.0/24) = endpoints utilisateurs
- VLAN 30 (10.0.30.0/29) = management — INTERDIT aux utilisateurs
- Outils internes accessibles sur *.charif-labs.tech (Portainer, Wazuh, n8n)
- Heures de travail normales : 07h00-21h00 (fuseau Europe/Paris)

Porte une attention PARTICULIÈRE à :
- Tout accès mémoire LSASS (Sysmon Event ID 10) — credential dumping potentiel
- Plus de 5 échecs d'authentification en moins de 3 minutes — brute force
- Connexion réseau d'un processus inattendu vers l'extérieur (Sysmon Event 3)
- Modification du registre Run/RunOnce — persistence probable
- Accès aux services internes en dehors des heures 07h-21h heure locale
- Login détecté sur un compte qui a été désactivé dans Keycloak
- Script PowerShell exécuté en tant que SYSTEM ou avec -EncodedCommand

Réponds UNIQUEMENT en JSON valide, sans texte avant ou après :
{
  "risk": "normal|suspect|critique",
  "confidence": 0.0,
  "reason": "explication courte en français",
  "ioc": ["liste des indicateurs observés"],
  "action_suggested": "recommandation pour l'équipe IT"
}
```

---
## Configuration Wazuh → n8n

```xml
<!-- /var/ossec/etc/ossec.conf -->

<!-- Alertes critiques : envoi immédiat -->
<integration>
  <n>custom-n8n-critical</n>
  <hook_url>https://n8n.charif-labs.tech/webhook/wazuh-critical</hook_url>
  <level>12</level>
  <alert_format>json</alert_format>
</integration>

<!-- Alertes moyennes : batch pour analyse IA -->
<integration>
  <n>custom-n8n-ai-batch</n>
  <hook_url>https://n8n.charif-labs.tech/webhook/wazuh-ai-batch</hook_url>
  <level>7</level>
  <max_level>11</max_level>
  <alert_format>json</alert_format>
</integration>
```

---

## Estimation des Coûts LLM

> Avec Claude API Haiku et pré-filtrage correct :

| Paramètre | Valeur |
|---|---|
| Alertes niveau 7-11 / jour (estimé) | ~200 |
| Après déduplication (5 min) | ~40 batches |
| Tokens par batch (20 alertes) | ~1500 input + 200 output |
| Coût Haiku (input $0.80/M, output $4/M) | ~5 centimes/jour |
| **Budget mensuel estimé** | **< 2€/mois** |

---
## Règles Anti-Spam IA

```javascript
// Function Node n8n : pré-filtrage avant appel LLM
const alerts = $input.body.alerts;

// 1. Dédupliquer : même rule_id + même host + < 5 minutes = garder 1 seul
const seen = new Set();
const deduplicated = alerts.filter(a => {
  const key = `${a.rule.id}-${a.agent.name}-${Math.floor(new Date(a.timestamp).getTime() / 300000)}`;
  if (seen.has(key)) return false;
  seen.add(key);
  return true;
});

// 2. Limiter à 20 alertes max par appel
const limited = deduplicated.slice(0, 20);

// 3. Formater de façon concise
return limited.map(a => ({
  time: a.timestamp,
  host: a.agent.name,
  rule: a.rule.id,
  desc: a.rule.description,
  level: a.rule.level
}));
```
