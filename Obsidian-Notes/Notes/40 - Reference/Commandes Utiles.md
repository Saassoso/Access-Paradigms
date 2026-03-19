---
tags: [référence, commandes, cheatsheet]
---
## Docker

```bash
docker compose ps                          # état des services
docker compose logs -f [service]           # logs temps réel
docker compose restart [service]           # redémarrer
docker compose up -d                       # démarrer en background
docker compose down                        # stopper (garde les volumes)
docker system df                           # espace disque
docker stats                               # CPU/RAM en temps réel
```

## Terraform

```bash
terraform init                             # initialiser
terraform plan                             # prévisualiser
terraform apply                            # appliquer
terraform output -raw [nom]                # récupérer une valeur
terraform state list                       # lister les ressources
```

## Ansible

```bash
ansible windows -m win_ping                # tester connectivité
ansible-playbook site.yml                  # lancer playbook principal
ansible-vault encrypt_string 'val' --name 'key'  # chiffrer une valeur
echo $VAULT_PASS | ansible-playbook --vault-password-file /dev/stdin playbook.yml
```

## OPNsense / Réseau

```bash
# Vérifier les règles firewall (depuis OPNsense shell)
pfctl -sr

# Depuis Docker Host — tester DNS
nslookup auth.charif-labs.tech
dig auth.charif-labs.tech CNAME

# Vérifier le tunnel Cloudflare
docker compose logs cloudflared | grep "Connected"
```

## Wazuh

```bash
# Vérifier l'état du manager
docker compose logs wazuh-manager | tail -50

# Depuis un agent Windows (PowerShell admin)
Get-Service WazuhSvc
```

## Windows (PowerShell admin)

```powershell
# Vérifier Entra Join
dsregcmd /status

# Vérifier WinRM
winrm enumerate winrm/config/listener

# Forcer sync politique (si Intune/MDM)
Start-Process "C:\Windows\System32\deviceenroller.exe" -ArgumentList "/c /AutoEnrollMDM"

# Vérifier BitLocker
manage-bde -status C:

# Voir events Sysmon
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 20
```

## Git

```bash
git status --short                         # état rapide
git log --oneline -10                      # 10 derniers commits
git tag v0.1-phase1-complete               # taguer une version
git diff --cached | grep -i secret         # vérifier avant push
```