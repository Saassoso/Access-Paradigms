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