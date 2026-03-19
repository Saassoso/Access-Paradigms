---

tags: [outil, iac, provisioning]

concepts: [IaC, DNS Delegation, Zero Trust Tunnel]

---
## Terrafrom 
## Rôle dans ce projet

Provisionne le **Control Plane** :
- Tunnel Cloudflare (UUID + secret)
- Enregistrements DNS CNAME proxifiés
- Enregistrements de vérification (TXT Microsoft)

Ne provisionne pas les VMs (VMware est local, pas d'API Terraform disponible).

## Structure du repo

```
terraform/
├── provider.tf          # Provider cloudflare ~> 4.0, version >= 1.5.0
├── variables.tf         # cloudflare_api_token, domain_name, account_id, zone_id
├── main.tf              # Tunnel + CNAME records + TXT records
├── ingress.tf           # Règles de routage ingress (hostname → service Docker)
└── .terraform.lock.hcl  # Provider verrouillé à 4.52.5
```

## Workflow

```bash
# 1. Initialiser (télécharge les providers)
terraform init

# 2. Prévisualiser les changements
terraform plan

# 3. Appliquer
terraform apply

# 4. Récupérer le tunnel token (pour cloudflared)
terraform output -raw cloudflare_zero_trust_tunnel_cloudflared_token
```

## State file

Le `terraform.tfstate` contient l'état réel de l'infrastructure (et des secrets !).  
→ **Ignoré par .gitignore** — ne jamais committer.  
→ Backup local obligatoire.

## Variables sensibles

```bash
# Ne jamais mettre dans terraform.tfvars
# Injecter via variables d'environnement
export TF_VAR_cloudflare_api_token="..."
export TF_VAR_cloudflare_account_id="..."
export TF_VAR_cloudflare_zone_id="..."
```

## Provider Cloudflare 4.x — resources clés

```hcl
# Tunnel
resource "cloudflare_zero_trust_tunnel_cloudflared" "sovereign_tunnel" {
  account_id = var.cloudflare_account_id
  name       = "sovereign-stack-tunnel"
  secret     = random_id.tunnel_secret.b64_std
}

# CNAME record
resource "cloudflare_record" "tunnel_cnames" {
  for_each = toset(["auth", "wazuh", "trmm", "n8n"])
  zone_id  = data.cloudflare_zone.main.id
  name     = each.key
  content  = "${cloudflare_zero_trust_tunnel_cloudflared.sovereign_tunnel.id}.cfargotunnel.com"
  type     = "CNAME"
  proxied  = true
}
```
