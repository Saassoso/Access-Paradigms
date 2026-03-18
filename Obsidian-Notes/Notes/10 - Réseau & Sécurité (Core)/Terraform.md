Écriture des fichiers `.tf` pour les Tunnels et le DNS.
## Terraform
## CHARIF-LABS-INFRA
### terrafrom
```
charif-labs-infra
/terrafrom
- provider.tf
- variables.tf
- main.tf
```
#### Cloudflare 


#### provider.tf
- Exige la version `~> 4.0` du provider `cloudflare/cloudflare`. Renseigne le bloc `provider "cloudflare"` pour qu'il consomme la variable `var.cloudflare_api_token`
