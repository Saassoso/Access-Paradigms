---

tags: [outil, infrastructure, containers]

concepts: [IaC]

---
## Rôle dans ce projet
Tous les services du plan de gestion tournent dans Docker sur le Docker Host (10.0.10.2) :
Authentik, cloudflared, Wazuh, n8n, Grafana, Prometheus, Uptime Kuma.
## Installation sur Ubuntu 24.04

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install docker.io docker-compose-v2 -y
sudo usermod -aG docker $USER
newgrp docker  # recharger le groupe sans déconnecter
```
## Réseau partagé sovereign_net

Le réseau `sovereign_net` est déclaré dans le docker-compose.yml principal et partagé entre les stacks via `external: true`.

```yaml
# docker/docker-compose.yml
networks:
  sovereign_net:
    driver: bridge
    name: sovereign_net
```

```yaml
# dans chaque stack enfant
networks:
  sovereign_net:
    external: true  # référence le réseau existant
```

Grâce à ce réseau partagé, `cloudflared` résout `authentik-server` par son nom de conteneur.
## Règle critique — OpenSearch RAM

```yaml
opensearch:
  environment:
    - "OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g"
```
Sans ce cap, OpenSearch consomme 4+ Go et crashe le PC.
## Volumes nommés externes
Les données persistantes (PostgreSQL Authentik, Wazuh indices) utilisent des volumes nommés déclarés comme `external: true` pour survivre aux `docker compose down`.
```bash
# Créer avant le premier démarrage
docker volume create authentik_database
docker volume create authentik_redis
```
## Commandes courantes
```bash
docker compose ps                    # état de tous les services
docker compose logs -f [service]     # logs en temps réel
docker compose restart [service]     # redémarrer un service
docker compose up -d                 # démarrer en background
docker system df                     # espace disque utilisé
```

# Docker 

Namesapces

Cgroups 

### Lighter VMs on non Linux machines and 
Kernel sharing 
### Container VS Image
- Running image 

### Image = Stack of layers
Application code
dependencies
runtime system libraries
base os

Docker file 
instractuin add Copy run 
- add a new layer
cmd env expose 
- add metadata no layers 

```
from node:20-alpine
workdire /app
copy package.json .
run npm install 
copy ..
```

cached layers // order of instrctuions to cache npm layers 
### Docker registry 
image distrubution 
shared  layer / layer cache 
### DockerFile 
**ENV**
- set env variable , at build and runtime
EXPOSE
- documentation . the port still need to be opened with -p flag
CMD
- default commande that run when container start (exe when container crated m not during build )
ENTRYPOINT 
- define executable m while CMD define the default arguments
	- can be overriden expliclity with --entrypoint 
### The build Conctext
docjker build .

- .dockerignore 
### Volumes 
**named volumes** 
- docker manage storage location , for persisten t data
`docker run -v my-data:/var/lib/postgresql/data postgres`
First-use copy

BIND MOUNTS
- local directory is mapped into the container 
`docker run -v ./src:/app/src my-app`
without bind and sync live into container , full rebuild nedded on evry chnage
Obscure ehat is the path into the image 
covverd bby the mount

### Networking
Container are isolated 
#### Host to container 
create a bridge `host:container` map to a port on machine 
- `docker run -p 3000:3000 my-app`
#### Continare to contanier 
docker create a default network : **bridge**
- container on same network reach other by ip @ 

Custom network docker offer dns resolution , by contianer name automaticcly 
- docker compose create a custome netwokr behind the see m whic make the docker netwk find each other 
```
docker network create my-network

docker run -- name api --network my-network my-api
docker run -- name db --network my-network postgres
```

Isolation : 3 tier architecture 


### Docker compose 
depends on 
- only stratup not readiness 
```
depends_on: 
db:
condition: service_healty
```

```
while(!connected)
retry with backoff
```

declarative yaml file to imperatiive commands 