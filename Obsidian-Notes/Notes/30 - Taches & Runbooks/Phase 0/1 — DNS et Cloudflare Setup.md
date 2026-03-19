---

tags: [tâche, phase-0, dns, cloudflare, terraform]

outils: [Cloudflare, Terraform]

concepts: [DNS Delegation, Zero Trust Tunnel]

statut: ✅ complété

---
## Contexte

Délégation DNS de `charif-labs.tech` vers Cloudflare.  
Provisioning du tunnel ZTNA et des CNAME records via Terraform.

## Étapes complétées

### 1. Désactivation DNSSEC (registrar Tech Domains)

Obligatoire avant de changer les NS pour éviter le rejet cryptographique.

### 2. Création compte Cloudflare + import domaine

Cloudflare → Add Site → charif-labs.tech  
Cloudflare fournit : `jasmine.ns.cloudflare.com` + `rohin.ns.cloudflare.com`

### 3. Remplacement NS chez Tech Domains

Anciens NS remplacés par les deux NS Cloudflare.  
Propagation vérifiée avec :
```bash
nslookup -type=ns charif-labs.tech
```

### 4. Token API Cloudflare

Cloudflare → My Profile → API Tokens → Create Token  
Permissions : Zone:DNS:Edit + Zone:Zone:Read

### 5. Terraform init + apply

```bash
export TF_VAR_cloudflare_api_token="..."
export TF_VAR_cloudflare_account_id="..."
export TF_VAR_cloudflare_zone_id="..."

cd terraform/
terraform init   # provider Cloudflare v4.52.5
terraform plan   # vérifier les ressources à créer
terraform apply  # créer tunnel + 4 CNAME + TXT Microsoft
```

Ressources créées :
- Tunnel `sovereign-stack-tunnel` (UUID généré)
- 4 CNAME proxifiés : auth, wazuh, trmm, n8n → `<uuid>.cfargotunnel.com`
- TXT `ms.charif-labs.tech` → `MS=ms76330167`

### 6. Récupérer le tunnel token

```bash
terraform output -raw cloudflare_zero_trust_tunnel_cloudflared_token
# Stocker dans docker/core-identity/cloudflared/.env
# TUNNEL_TOKEN=<valeur>
```

### 7. Déployer cloudflared

```bash
cd docker/
docker compose up -d cloudflared
docker compose logs cloudflared  # vérifier "Connected to Cloudflare"
```

### 8. Déployer Authentik

```bash
docker volume create authentik_database
docker volume create authentik_redis
cd docker/core-identity/
docker compose up -d
```

Vérifier : `https://auth.charif-labs.tech` → page de login Authentik ✅

## Validation ✅

```bash
# Tunnel actif
docker compose logs cloudflared | grep "Connected"

# DNS propagé
dig auth.charif-labs.tech CNAME

# Authentik accessible
curl -I https://auth.charif-labs.tech
```

## Entra ID — vérification domaine

Entra Admin → Custom domain names → charif-labs.tech  
Type : TXT, Valeur : `MS=ms76330167`  
Statut : Verified ✅
