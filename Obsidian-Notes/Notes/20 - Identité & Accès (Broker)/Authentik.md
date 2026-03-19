- Configuration de compte Admin initial : `akadmin`
### authentik 
![](images/Authentik.png)

Access Admin Interface
![](images/Authentik-2.png)
- Create Roles with permissions :

| Nom       | Parent    | Rôle                  |
| --------- | --------- | --------------------- |
| `Bureau1` | _(aucun)_ | OU racine Bureau 1    |
| `Basique` | `Bureau1` | Utilisateurs standard |
| `Bureau2` | _(aucun)_ | OU racine Bureau 2    |
| `Admins`  | `Bureau2` | Administrateurs       |
![](images/Authentik-1.png)
- Create Groups with Roles : 
![](images/Authentik-3.png)
- Create Users and put them in a folder :
![](images/Authentik-4.png)
- Assign user to Groups :
![](images/Authentik-5.png)

#### Test Login
![](images/Authentik-6.png)

