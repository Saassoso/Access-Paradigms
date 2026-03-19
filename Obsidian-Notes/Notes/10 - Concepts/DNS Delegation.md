---

tags: [concept, dns, réseau]

liens: [Cloudflare]

---
## Qu'est-ce que la délégation DNS ?

Transférer l'autorité de résolution d'une zone DNS à un autre serveur de noms. Concrètement : dire au registrar "les réponses pour mondomaine.com, c'est Cloudflare qui les donne".

## Hiérarchie DNS

![Hiérarchie DNS](Canvas/Hiérarchie%20DNS.canvas)
## Les enregistrements clés

| Type | Rôle | Exemple dans ce projet |
|---|---|---|
| NS | Délègue la zone à des serveurs de noms | `charif-labs.tech NS jasmine.ns.cloudflare.com` |
| A | IP IPv4 d'un hôte | `auth.charif-labs.tech A <IP>` |
| CNAME | Alias vers un autre nom | `auth.charif-labs.tech CNAME <tunnel-uuid>.cfargotunnel.com` |
| TXT | Vérification de propriété | `ms.charif-labs.tech TXT MS=ms76330167` |

## DNSSEC et pourquoi le désactiver avant de changer les NS

**DNSSEC** signe cryptographiquement les enregistrements DNS. Si tu changes les NS alors que **DNSSEC** est actif avec les anciennes clés, les résolveurs rejettent les réponses (signature invalide). Ordre correct :
1. Désactiver DNSSEC chez l'ancien registrar
2. Changer les NS vers Cloudflare
3. Attendre la propagation (48h max)
4. Réactiver DNSSEC chez Cloudflare si voulu

## Dans ce projet

Registrar : **Tech Domains**  
Zone DNS déléguée à : [[../20 - Outils/Cloudflare]]  
Provisioning DNS : [[../../Terraform]] (provider Cloudflare)

Enregistrements gérés par Terraform :
- 4 CNAME proxifiés (auth, wazuh, trmm, n8n)
- TXT de vérification Microsoft (`ms.charif-labs.tech`)
- A placeholder Google (`google.charif-labs.tech`) — en attente résolution blocage

## Vérifier la propagation
```bash
nslookup -type=ns charif-labs.tech
dig +trace auth.charif-labs.tech
```

![](images/DNS%20Delegation.png)