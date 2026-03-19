| Composant         | Rôle IAM explicite                                            |
| ----------------- | ------------------------------------------------------------- |
| Authentik         | IdP — PDP central — émet les assertions d'identité            |
| Cloudflare Access | PEP — applique les décisions d'accès à l'edge                 |
| n8n               | Moteur JML — automatise les transitions Joiner/Mover/Leaver   |
| Action1           | Enforcement sur le device — étend le PEP jusqu'à l'endpoint   |
| Ansible           | PAP — définit et applique les politiques d'accès (CIS, LAPS)  |
| Wazuh             | Audit trail — prouve que les décisions IAM ont été appliquées |
| Entra ID          | SP fédéré + PAM break-glass                                   |