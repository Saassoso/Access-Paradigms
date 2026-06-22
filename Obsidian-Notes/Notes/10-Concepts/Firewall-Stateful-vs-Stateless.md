---

tags: [concept, sécurité, réseau, firewall]

liens: [OPNsense]

---
## Firewall stateless
Évalue chaque paquet **indépendamment**. Une règle `ALLOW TCP src:A dst:B port:443` autorise les paquets dans ce sens. Pour les réponses, il faut une règle symétrique.
Problème : difficile à gérer, ouverture involontaire de ports.
## Firewall stateful
Track les **connexions** TCP/UDP dans une table d'état. Une règle `ALLOW src:A dst:B port:443` autorise la connexion entière — les réponses (`B → A`) sont automatiquement autorisées car le firewall sait qu'elles font partie d'une connexion existante.
```
Table d'état :
src:10.0.20.10:52341  dst:8.8.8.8:53  proto:UDP  state:ESTABLISHED
→ La réponse DNS de 8.8.8.8 est autorisée automatiquement
```
## Default deny
Principe fondamental du zero trust : **tout est bloqué par défaut**, sauf ce qui est explicitement autorisé. 
OPNsense applique ce principe — la dernière règle implicite est `DENY ALL`.
## Dans ce projet (OPNsense)
Deux règles sur l'interface VLAN 20 (dans l'ordre d'évaluation) :
```
Règle 1 : BLOCK — VLAN20 net → LAN net (Management)
Règle 2 : PASS  — VLAN20 net → any (Internet)
```

> **Ordre critique** : OPNsense évalue les règles de haut en bas et s'arrête à la première correspondance. La règle BLOCK doit être **avant** la règle PASS, sinon le trafic vers Management passerait par la règle "any".
## Zenarmor — extension NGFW L7
[[../../Zenarmor]] ajoute une inspection au-delà de L4 (IP/port) — il peut filtrer par application, catégorie de domaine, contenu. 
Un firewall stateful classique voit `TCP port 443` ; Zenarmor voit `Netflix HTTPS streaming`.