---

tags: [outil, versionning, iac]

concepts: [IaC]

---
# Rôle dans ce projet

Source de vérité unique pour toute l'infrastructure. Si c'est pas dans Git, ça n'existe pas officiellement.

## Repos

| Repo | Contenu | Visibilité |
|---|---|---|
| `charif-labs-infra` | Terraform, Docker, Ansible | Privé |
| `access-paradigms` | Notes Obsidian | Privé |

## .gitignore — règles critiques

```gitignore
# Terraform — state et secrets
*.tfstate
*.tfstate.*
*.tfvars
*.tfvars.json
**/.terraform/*
crash.log

# Secrets
.env
*.key
*.pem
vault_pass
```

## Workflow quotidien

```bash
git status --short          # vérifier qu'aucun secret ne traîne
git add ansible/ docker/    # ajouter par dossier, pas git add .
git commit -m "feat: deploy wazuh manager + opensearch cap"
git push
```

## Convention de commits

```
feat: nouvelle fonctionnalité
fix: correction de bug
chore: maintenance (update deps, cleanup)
docs: documentation
```

## Tags — marquer les étapes

```bash
git tag v0.0-phase0-complete
git tag v0.1-phase1-ztna-deployed
git push --tags
```

## Vérification avant push

```bash
# S'assurer qu'aucun secret ne part
git diff --cached | grep -i "password\|secret\|token\|key"
```
