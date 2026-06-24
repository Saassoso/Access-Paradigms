---

tags: [concept, iac, devops, terraform, ansible]

liens: [Terraform, Ansible, Git]

---
## Définition
Décrire l'infrastructure dans du **code versionné** plutôt que dans des clics UI. Tout ce qui peut être configuré manuellement peut être codé et rejoué.
## Principes fondamentaux
### Idempotence
Exécuter le même code 10 fois donne le même résultat. Le code vérifie l'état actuel avant d'agir.
```
Ansible — idempotent : "si le paquet est déjà installé, ne rien faire"
Script bash — pas idempotent par défaut : "installer le paquet" (échoue si déjà là)
```
### Déclaratif vs Impératif

| Paradigme | Approche | Outils |
|---|---|---|
| **Déclaratif** | Décrire l'état final souhaité | [[../../Terraform]], Docker Compose, Ansible |
| **Impératif** | Décrire les étapes pour y arriver | Scripts bash, PowerShell |
### Immutable infrastructure
Au lieu de modifier un serveur existant, on crée un nouveau serveur avec la config souhaitée et on détruit l'ancien. Simplifie le rollback.
## Les deux outils IaC de ce projet

| | [[../../Terraform]] | [[../20-Outils/Ansible]] |
|---|---|---|
| Rôle | **Provisioning** — créer les ressources | **Configuration** — configurer ce qui existe |
| Quand | Avant que la ressource existe | Après que la ressource existe |
| State | Oui (tfstate) | Non (vérifie l'état à chaque run) |
| Exemple | Créer le tunnel Cloudflare, les DNS | Installer Docker, configurer CIS hardening |
## Git comme source de vérité
Tout le code IaC est dans [[../../Git]]. Le repo = la documentation réelle de l'infrastructure. Si c'est pas dans Git, ça n'existe pas officiellement.
```
sovereign-stack/
├── terraform/    ← ce qui EST (Cloudflare)
├── ansible/      ← ce qui est CONFIGURÉ (endpoints)
├── docker/       ← ce qui TOURNE (services)
└── .gitignore    ← *.tfstate, .env, *.tfvars ne sont JAMAIS committés
```
## Rule #1 — Jamais de secrets dans Git
```gitignore
# .gitignore obligatoire
*.tfstate
*.tfstate.*
*.tfvars
*.tfvars.json
.env
*.key
crash.log
```
