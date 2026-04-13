**Problème :** 
- User.ReadWrite.All + Directory.ReadWrite.All
	- `donne accès à tous les utilisateurs du tenant, y compris les admins.

**Correction — Permissions minimales :**
```
User.ReadWrite.All     → Remplacer par User.ReadWrite (scope limité)
Directory.ReadWrite.All → Supprimer (non nécessaire)
Ajouter : User.EnableDisableAccount.All (si disponible)
```

> Si `User.ReadWrite` ne suffit pas pour le provisioning, documenter explicitement pourquoi `User.ReadWrite.All` est requis et limiter via **Conditional Access App** à un groupe d'utilisateurs cibles.