---

tags: [concept, réseau, ztna, cloudflare]

liens: [Zero Trust, Cloudflare]

---
## Le problème que ça résout

Pour exposer un service interne sur internet, la méthode classique = ouvrir un port sur le routeur (NAT/port forwarding). Problème : chaque port ouvert est une surface d'attaque directe.

## Le pattern ZTNA Tunnel

Au lieu d'ouvrir des ports **en entrée**, on établit une connexion **sortante** depuis le serveur vers un edge provider (Cloudflare, Ngrok, Tailscale...). Le trafic entrant transite par cette connexion inversée.

![Untitled](../99%20-%20Attachment/Canvas/ZTNA%20Tunnel.canvas)
## Comment ça fonctionne techniquement

1. Le daemon `cloudflared` tourne sur le serveur
2. Il établit une connexion persistante HTTPS vers le Cloudflare Edge
3. Cloudflare reçoit les requêtes des clients sur le domaine configuré
4. Cloudflare les route vers le serveur via la connexion existante
5. La réponse repart par le même chemin

## Avantages

- **Zéro port exposé** — pas de NAT, pas de firewall inbound à gérer
- **TLS automatique** — Cloudflare gère le certificat HTTPS
- **DDoS protection** incluse (Cloudflare absorbe le trafic)
- **Pas d'IP publique nécessaire** — fonctionne même derrière un CGNAT

## Implémentation dans ce projet

Outil : [[../20 - Outils/Cloudflare]] Tunnel  
Daemon : `cloudflared` dans Docker  
Provisioning : [[../../Terraform]] (tunnel UUID + CNAME records)

Services exposés :
- `auth.charif-labs.tech` → keycloak-server:9000
- `wazuh.charif-labs.tech` → wazuh-dashboard:443
- `trmm.charif-labs.tech` → tactical-rmm:443
- `n8n.charif-labs.tech` → n8n:5678

## Alternatives

| Outil             | Modèle                       | Gratuit      |
| ----------------- | ---------------------------- | ------------ |
| Cloudflare Tunnel | Edge provider commercial     | Oui          |
| Tailscale Funnel  | Mesh VPN + expose            | Oui (limité) |
| Ngrok             | Tunnel HTTP/TCP              | Oui (limité) |
| WireGuard         | VPN P2P (pas un tunnel ZTNA) | Oui          |