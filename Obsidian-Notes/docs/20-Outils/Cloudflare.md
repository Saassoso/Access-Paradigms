---

tags: [outil, ztna, dns, cdn]

concepts: [Zero Trust Tunnel, DNS Delegation]

---
## Rôle dans ce projet

Deux rôles distincts :
1. **DNS autoritatif** pour `charif-labs.tech` (délégation depuis Tech Domains)
2. **ZTNA Tunnel** pour exposer les services sans ouvrir de ports

Comprend [[Zero Trust Tunnel]] et [[DNS Delegation]] avant de configurer.

## Composants utilisés

| Composant | Rôle | Provisionné par |
|---|---|---|
| Zone DNS | Autorité DNS pour charif-labs.tech | [[../../Terraform]] |
| Tunnel `sovereign-stack-tunnel` | Connexion sortante cloudflared → Edge | [[../../Terraform]] |
| CNAME records (×4) | auth, wazuh, trmm, n8n | [[../../Terraform]] |
| TXT record | Vérification domaine Microsoft | [[../../Terraform]] |
| cloudflared daemon | Maintient la connexion tunnel | Docker Compose |

## Tunnel — architecture Docker

Le daemon `cloudflared` tourne dans Docker, connecté au réseau `sovereign_net`.  
Il résout les services par leur nom Docker (ex: `keycloak-server`).

```yaml
# docker/cloudflared/docker-compose.yml
cloudflared:
  image: cloudflare/cloudflared:latest
  command: tunnel --no-autoupdate run --token ${TUNNEL_TOKEN}
  networks:
    - sovereign_net
```

Le `TUNNEL_TOKEN` est l'output `cloudflare_zero_trust_tunnel_cloudflared_token` de [[../../Terraform]], stocké dans `.env` (ignoré par .gitignore).

## Token API

Permissions minimales : `Zone:DNS:Edit` + `Zone:Zone:Read`  
Stocké dans : variable d'environnement `TF_VAR_cloudflare_api_token`  
Ne jamais stocker dans un fichier `.tfvars` commité.

## Ingress rules (ingress.tf)

```hcl
ingress_rule { hostname = "auth.charif-labs.tech"  service = "http://keycloak-server:9000" }
ingress_rule { hostname = "wazuh.charif-labs.tech" service = "https://wazuh-dashboard:443" }
ingress_rule { service = "http_status:404" }  # catch-all obligatoire
```

## CNAME proxifié (orange cloud)

Le CNAME pointe vers `<tunnel-uuid>.cfargotunnel.com` avec le proxy Cloudflare activé.  
→ L'IP réelle du serveur n'est jamais exposée.  
→ Cloudflare gère le certificat TLS.

