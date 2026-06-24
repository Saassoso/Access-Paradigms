**Zero-Trust Network Access (ZTNA)** architecture
### Core Architectural Pillars
- **Identity-Centric Security:** Shifting from local/[[SAML]] accounts to a unified identity flow using **[[Phase 0/GCPW]] and **[[Phase 0/keycloak]]** as an [[OIDC]] broker to federate [[Google Cloud Identity]] and [[Microsoft Entra ID]].
- **Layer 7 Perimeter:** Replacing pfSense with **[[Phase 0/OPNsense]] + [[../../Zenarmor]]** for application-level visibility, paired with **[[Phase 0/Cloudflare]] Tunnels** to eliminate inbound open ports.
- **Automated Response:** Implementing event-driven workflows via **[[n8n]]** and **[[Wazuh]]** to achieve an automated "Hostile Offboarding" SLA of less than 5 seconds.
- **Infrastructure as Code (IaC):** Standardizing deployment and configuration through **[[../../Terraform]]**, **[[../20-Outils/Ansible]]**, and **[[Tactical RMM]]** to ensure "Zero-Touch" provisioning and CIS hardening.
### Le Flux de Télémétrie (XDR Multi-Cloud)
L'objectif est d'avoir une visibilité totale sur les postes physiques et les environnements cloud.
- **Ingestion Locale :** L'agent Wazuh, déployé sur l'Endpoint, remonte la configuration [[Sysmon ]](Event IDs 1, 3 et 10) pour analyser l'exécution des processus.
- **Ingestion Cloud :** L'API de Wazuh interroge simultanément le [[Microsoft Graph API]] et le [[Google Admin SDK]] pour récupérer les logs d'audit et les échecs de connexion des tenants cloud.
- **Corrélation :** Le dashboard Wazuh centralise simultanément l'activité système locale et les connexions cloud.
### Le Flux d'Automatisation et de Réponse (SOAR)
L'architecture abandonne les mécanismes de "polling" au profit d'une exécution immédiate basée sur les événements (Event-Driven).
- **Le Moteur :** n8n exécute des workflows déclenchés par des Webhooks entrants soumis à une validation HMAC stricte pour garantir l'absence de latence.
- **Runbook 1 (Hostile Offboarding) :** Lors de la soumission d'un payload JSON, n8n parallélise les requêtes : révocation des sessions [[SSO]] (keycloak), réinitialisation des mots de passe (Google/Graph API), rotation LAPS forcée et coupure réseau du poste via Tactical RMM. Le SLA cible l'exécution en moins de 5 secondes sans aucune interface graphique.
- **Runbook 2 (Réponse XDR) :** Si Wazuh détecte un "Credential Dumping" (accès à la mémoire LSASS, Sysmon Event ID 10), il déclenche un webhook n8n . n8n extrait le Hostname et commande à Tactical RMM de couper physiquement la carte réseau de l'endpoint infecté en moins de 60 secondes.


Docker connect to frontend and backend networks