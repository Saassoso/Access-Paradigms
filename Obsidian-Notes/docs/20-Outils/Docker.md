---

tags: [outil, infrastructure, containers]

concepts: [IaC]

---
## Rôle dans ce projet
Tous les services du plan de gestion tournent dans Docker sur le Docker Host (10.0.10.2) :
keycloak, cloudflared, Wazuh, n8n, Grafana, Prometheus, Uptime Kuma.
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

Grâce à ce réseau partagé, `cloudflared` résout `keycloak-server` par son nom de conteneur.
## Règle critique — OpenSearch RAM

```yaml
opensearch:
  environment:
    - "OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g"
```
Sans ce cap, OpenSearch consomme 4+ Go et crashe le PC.
## Volumes nommés externes
Les données persistantes (PostgreSQL keycloak, Wazuh indices) utilisent des volumes nommés déclarés comme `external: true` pour survivre aux `docker compose down`.
```bash
# Créer avant le premier démarrage
docker volume create keycloak_database
docker volume create keycloak_redis
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
## 1. Under the Hood: How Docker Works
Docker isn't magic; it relies on core Linux kernel features to create isolation without the heavy overhead of traditional Virtual Machines (VMs).

* **Namespaces:** Provide **isolation**. They ensure a container only sees its own processes, network, and file system.
* **Cgroups (Control Groups):** Provide **resource limitation**. They restrict how much CPU, memory, or disk I/O a container can use.
* **Kernel Sharing:** Unlike VMs that boot a full guest OS, containers share the host's Linux kernel. (On Windows/Mac, Docker runs a lightweight Linux VM in the background to provide this kernel).

---

## 2. Images vs. Containers
* **Image:** A static, read-only stack of layers. It is the blueprint.
* **Container:** A running instance of an image. It adds a thin, writable layer on top of the image stack.

### Anatomy of an Image (Stack of Layers)
An image is built from the bottom up:
1.  **Base OS** (e.g., Alpine, Ubuntu)
2.  **Runtime / System Libraries** (e.g., Node.js, Python)
3.  **Dependencies** (e.g., `node_modules`)
4.  **Application Code**

*Note on Registry:* Docker registries (like Docker Hub) distribute these images. They save bandwidth by sharing and reusing cached layers across different images.

---

## 3. The Dockerfile: Building the Blueprint
The `Dockerfile` contains the instructions to build an image. 

### Instructions & Layers
* **Layer-creating instructions:** `RUN`, `COPY`, `ADD`. Every time you use these, Docker creates a new layer.
* **Metadata instructions:** `CMD`, `ENV`, `EXPOSE`, `WORKDIR`. These do *not* create new layers; they just add configuration metadata to the image.

### Layer Caching (Crucial for Speed)
Docker caches layers to speed up builds. The **order of instructions matters**. You should copy dependencies and install them *before* copying your source code. 
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json .         # Copy only package.json first
RUN npm install             # This layer gets cached unless package.json changes
COPY . .                    # Now copy the frequently changing source code
```

### Key Commands Explained
| Command | What it does |
| :--- | :--- |
| **ENV** | Sets environment variables at both build time and runtime. |
| **EXPOSE** | Purely documentation. It tells the user which port the app uses, but you *still* need to map it at runtime with `-p`. |
| **CMD** | The default command executed when the container starts. Can be easily overridden by passing arguments to `docker run`. |
| **ENTRYPOINT** | Defines the main executable of the container. It is harder to override (requires `--entrypoint`). Often used alongside CMD (where CMD provides default arguments to the ENTRYPOINT). |

### The Build Context & Ignore
* `docker build .` sends everything in the current directory (`.`) to the Docker daemon as the "build context".
* **.dockerignore:** Used to prevent unnecessary files (like local `node_modules` or `.git`) from being sent to the daemon, speeding up the build and keeping the image lean.

---

## 4. Volumes & Storage (Data Persistence)
Containers are ephemeral; if they die, their writable layer dies. To save data, use storage.

### Named Volumes
* Managed entirely by Docker (stored in a hidden location on the host).
* Great for persistent data like database files.
* **First-use copy:** If the container has files at the volume's destination path, Docker will copy those files into the volume the first time it's created.
* *Example:* `docker run -v my-data:/var/lib/postgresql/data postgres`

### Bind Mounts
* Maps a specific directory on your local host directly into the container.
* Essential for local development (syncs live changes into the container without needing a full rebuild).
* **Warning:** The host directory completely obscures (covers up) whatever was at that path inside the image.
* *Example:* `docker run -v ./src:/app/src my-app`

---

## 5. Networking
Containers are isolated by default. 

* **Host to Container:** You must bridge the host to the container by mapping a port.
    * `docker run -p 3000:3000 my-app` (maps Host Port 3000 to Container Port 3000).
* **Container to Container:**
    * Docker puts containers on a default network called `bridge`. They can reach each other via IP address, but IPs change.
    * **Custom Networks:** Create one with `docker network create my-network`. Containers on a custom network get **automatic DNS resolution** (they can talk to each other using their container names instead of IPs).
    * *Example:* `docker run --name api --network my-network my-api` can talk to `docker run --name db --network my-network postgres` by simply calling `db:5432`.
* **Isolation:** Useful for a 3-tier architecture (e.g., front-end network, back-end network, database network) to strictly control which containers can see each other.

---

## 6. Docker Compose
Docker Compose translates a **declarative YAML file** into the **imperative commands** needed to spin up multiple containers, networks, and volumes at once. Under the hood, it automatically creates a custom network so your services can discover each other by name.

### Startup Order (`depends_on`)
By default, `depends_on` only waits for the dependency container to *start*, not for the application inside it to actually be ready (e.g., a database booting up).
To fix this, you use health checks:
```yaml
depends_on:
  db:
    condition: service_healthy
```
*Note:* Your application code should still be resilient! 
Always implement 
```
while (!connected) { retry with backoff }
``` 
logic in your code to handle database drops or slow startups gracefully.