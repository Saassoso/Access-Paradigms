---

tags: [outil, siem, xdr, sécurité]

concepts: []

---
## Rôle dans ce projet

SIEM/XDR open source. Centralise les logs, détecte les menaces, évalue la conformité CIS.

Flux principal : Sysmon (endpoint) → Agent Wazuh → Manager → Dashboard.

## Architecture
![Wazuh Architecture](../99%20-%20Attachment/Canvas/Wazuh%20Architecture.canvas)
## Déploiement Docker — cap RAM obligatoire

```yaml
opensearch:
  environment:
    - "OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g"
```

Sans ce cap : OpenSearch prend 4+ Go → crash garanti.

## Sysmon Event IDs critiques

| Event ID | Nom | Intérêt sécurité |
|---|---|---|
| 1 | Process Create | Quel processus a lancé quoi |
| 3 | Network Connection | Connexions TCP/UDP par processus |
| 10 | Process Access | Accès mémoire LSASS = credential dumping |
| 11 | File Create | Fichier créé par quel processus |
| 13 | Registry Value Set | Modification registre par processus |

Config recommandée : sysmon-modular (Olaf Hartong) — ne pas partir de zéro.

## SCA — Security Configuration Assessment

Module qui évalue la conformité CIS Benchmark.

```bash
# Lancer un scan manuel depuis le Dashboard
# Agents → [nom agent] → SCA → Run scan

# Target score : ≥ 85%
```

## FIM — File Integrity Monitoring

Surveillance des fichiers critiques. Alerte si modification détectée.

```xml
<!-- ossec.conf sur l'agent -->
<syscheck>
  <directories realtime="yes">C:\Windows\System32</directories>
  <directories realtime="yes">C:\Windows\System32\drivers</directories>
</syscheck>
```

## Webhook vers n8n

Wazuh peut envoyer des alertes vers un webhook [[n8n]] via les intégrations custom :

```xml
<!-- ossec.conf -->
<integration>
  <name>custom-n8n</name>
  <hook_url>https://n8n.charif-labs.tech/webhook/wazuh-alert</hook_url>
  <level>12</level>  <!-- niveau d'alerte minimum -->
  <alert_format>json</alert_format>
</integration>
```
