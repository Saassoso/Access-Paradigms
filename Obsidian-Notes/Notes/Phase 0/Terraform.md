Écriture des fichiers `.tf` pour les Tunnels et le DNS.
## Terraform
## CHARIF-LABS-INFRA

### Cloudflare 
#### Terminus
Crée un dossier pour ton tunnel :
- `mkdir -p ~/cloudflare && cd ~/cloudflare`
Crée le fichier de configuration :
- `nano docker-compose.yml`
``` yaml 
services:
  tunnel:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared-tunnel
    restart: unless-stopped
    command: tunnel run
    environment:
      - TUNNEL_TOKEN=TON_SUPER_LONG_TOKEN_ICI
```
Allumer le moteur :
- `docker compose up -d`

```
charif-labs-infra
/cloudflare
- provider.tf
- variables.tf
- main.tf
```

#### provider.tf
- Exige la version `~> 4.0` du provider `cloudflare/cloudflare`. Renseigne le bloc `provider "cloudflare"` pour qu'il consomme la variable `var.cloudflare_api_token`
