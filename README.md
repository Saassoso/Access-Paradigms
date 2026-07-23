# Access-Paradigms

```
access-paradigms/
├── 📂 Notes/
│   ├── 📂 00 - Architecture & Management/
│   │   ├── Sovereign Security Stack.md  # Le concept global ZTNA / SOAR
│   │   ├── Plan d'Action.md             # Le suivi d'exécution
│   │   └── Prerequisites.md             # Les prérequis techniques
│   ├── 📂 10 - Réseau & Sécurité (Core)/
│   │   ├── OPNsense.md                  # Topologie L2/L3 et Firewall
│   │   ├── Network Namespaces.md        # Lab testing de l'isolation
│   │   ├── netplan.md                   # Configuration IP de l'hôte Linux
│   │   └── Zenarmor.md                  # Le futur NGFW Layer 7
│   ├── 📂 20 - Identité & Accès (Broker)/
│   │   ├── Authentik.md                 # Configuration IdP et OIDC
│   │   ├── Cloudflare.md                # Tunnels et DNS (Délégation)
│   │   └── GCPW.md                      # Intégration identité sur Windows
│   ├── 📂 30 - Systèmes & IaC/
│   │   ├── Docker.md                    # Déploiement moteur et hygiène
│   │   ├── Terraform.md                 # Structure de l'IaC Cloudflare
│   │   ├── Git.md                       # Gestion de versionning
│   │   └── Termius.md                   # Tunnels SSH pour administration
│   ├── 📂 40 - Endpoints/
│   │   └── Windows.md                   # Configuration de l'hyperviseur et OS
│   └── 📂 images/                       # Dossier contenant tes captures (OPNsense, etc.)
├── .gitignore                           # Ignore le dossier système .obsidian
└── README.md
```
---

# Access Paradigms Notes (Obsidian Export)

This section details the contents of the `Obsidian-Notes` directory, which contains a collection of notes, concepts, runbooks, and task-related documentation. These notes were originally maintained in Obsidian and have been refactored for public presentation on GitHub.

The content is organized within the `Obsidian-Notes/docs` directory.

## Structure

The `docs` directory contains the following subdirectories:

*   **00-Project**: General project-related documentation.
*   **10-Concepts**: Core concepts and theoretical explanations.
*   **20-Outils**: Tools and utilities documentation.
*   **30-Runbooks**: Operational runbooks and procedures.
*   **40-Journal**: Journal entries or time-based notes.
*   **50-Tasks**: Task-specific notes.
*   **99-Attachments**: Contains various attachments and media files linked within the notes.
*   **Views**: Specific views or filtered sets of notes.
*   **HOME.md**: A good starting point for navigating the notes.

## Infrastructure

The infrastructure for this project is managed in a separate repository: [charif-labs-infra](https://github.com/charif-labs-org/charif-labs-infra)

## Usage

To explore the notes, simply browse the `Obsidian-Notes/docs` directory. Start with `HOME.md` for an overview.
