### Configuration du Tunnel dans Termius
1. **Création de l'hôte (Host) :** Ajoute ton serveur Ubuntu dans Termius en utilisant l'adresse IP **NAT** (`192.168.138.x`) que tu viens de noter, avec tes identifiants (utilisateur `mgmt` et ton mot de passe).
![](../99%20-%20Attachment/images/2026-03-16%2008_26_07-Termius%20-%20Hosts.png)
2. **Configuration du Port Forwarding :**
    - Dans Termius, va dans l'onglet **Port Forwarding**.
    - Ajoute une nouvelle règle de type **Local**.
    - **Local port (Port local) :** `8444` (ou n'importe quel port libre sur ton PC physique).
    - **Destination address (Adresse de destination) :** `10.0.10.1` (L'IP de la WebGUI OPNsense sur le réseau Management).
    - **Destination port (Port de destination) :** `443` (Le port HTTPS d'OPNsense).
    - **Select Host :** Sélectionne l'hôte Ubuntu que tu viens de créer à l'étape 1.
![](../99%20-%20Attachment/images/2026-03-16%2008_25_56-Terminus-Port-Forwarding.png)
3. **Activation :** Démarre le tunnel (double-clique sur la règle ou clique sur l'icône de connexion).
### Phase 3 : Accès au Pare-feu

Une fois le tunnel Termius actif, ouvre ton navigateur web préféré sur ta **machine physique** et navigue vers l'adresse locale : `https://127.0.0.1:8443`