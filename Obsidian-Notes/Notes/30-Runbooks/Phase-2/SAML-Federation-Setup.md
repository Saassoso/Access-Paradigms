## Objectif du Scénario
Permettre aux utilisateurs de s'authentifier sur des postes Windows 11 (VLAN 20) joints à Entra ID, en utilisant exclusivement leurs identifiants gérés par Keycloak.
* **Identity Provider (IdP) :** Keycloak
* **Service Provider (SP) :** Microsoft Entra ID
* **Protocole :** SAML 2.0

---

## Étape 1 : Le Piège Architectural (Entra ID : Enterprise App vs Domain Federation)
* **Action initiale :** Tentative de configuration de la relation SAML via le menu "Enterprise Applications" dans le portail Entra ID.
* **Le Problème :** Le portail a généré un "Token signing certificate" et une "Login URL" pour Entra.
* **Explication technique :** Le menu *Enterprise Applications* sert à faire d'Entra ID l'IdP pour des applications tierces (SaaS). Si l'on configure SAML ici, Entra s'attend à *émettre* des jetons, et non à en recevoir. 
* **La Résolution :** Pour forcer Entra ID à agir comme SP et déléguer l'authentification Windows, il faut configurer une **Domain Federation** via le Microsoft Graph API (PowerShell). Cette option est intentionnellement masquée dans l'interface graphique de Microsoft.

---

## Étape 2 : Préparation de l'IdP (Keycloak)
Avant de lancer PowerShell, Keycloak doit être prêt à recevoir les requêtes de Microsoft.

* **Configuration du SAML Provider :**
  * **ACS URL :** `https://login.microsoftonline.com/login.srf` (Point de terminaison universel de Microsoft).
  * **Issuer (EntityID) :** `https://auth.charif-labs.tech`
  * **Service Provider Binding :** `Post`
  * **Audience :** `urn:federation:MicrosoftOnline`
  * **Property Mappings :** Sélection de `UPN`, `Email`, et `Groups`.
  * **NameID Property Mapping :** `Keycloak default SAML Mapping: Email` (ou UPN).

* **Le Piège du Certificat :**
  * **Action :** Exportation du certificat public "Keycloak default self-signed".
  * **Explication technique :** PowerShell n'accepte pas les fichiers `.pem` ou `.crt` standard. Il exige une chaîne Base64 continue.
  * **Résolution :** Ouverture du certificat dans un éditeur de texte, suppression des balises `-----BEGIN CERTIFICATE-----` et `-----END CERTIFICATE-----`, et **suppression de tous les retours à la ligne** pour obtenir une seule chaîne de texte ininterrompue.

---

## Étape 3 : Exécution de la Fédération via PowerShell
Cette étape a nécessité de surmonter plusieurs obstacles liés à l'environnement PowerShell et aux exigences strictes de Microsoft.

### Sous-étape 3.1 : Connexion au Graph API & Compte Break-Glass
* **Commande :** `Connect-MgGraph -Scopes "Domain.ReadWrite.All", "Directory.AccessAsUser.All"`
* **Erreur rencontrée :** `AADSTS900023: Specified tenant identifier 'y' is neither a valid DNS name...`
* **Explication technique :** Une touche "y" (probablement tapée pour accepter l'installation du module PSGallery) a été mémorisée par le terminal et injectée dans la commande de connexion, corrompant l'ID du tenant envoyé à Microsoft.
* **Résolution & Bonne pratique :** Fermeture du pop-up, nettoyage du terminal. Reconnexion en utilisant un compte administrateur **"Break-Glass"** natif (`admin@<tenant>.onmicrosoft.com`). *Pourquoi ?* Si la fédération casse l'authentification du domaine personnalisé (`@ms.charif-labs.tech`), le compte `.onmicrosoft.com` ne sera jamais redirigé vers Keycloak, garantissant toujours un accès de secours au tenant.

### Sous-étape 3.2 : Le Script de Fédération et la Règle MFA
* **Commande initiale préparée :** Exécution de `New-MgDomainFederationConfiguration` avec les variables `$issuerUri`, `$logOnUri`, `$logOffUri` et `$cert`.
* **Erreur rencontrée :** `FederatedIdpMfaBehavior cannot be empty (Status: 400 BadRequest)`
* **Explication technique :** Récemment, Microsoft a durci ses règles de sécurité Zero Trust. Lors d'une fédération, Entra ID exige de savoir explicitement qui est responsable du MFA pour éviter une faille de sécurité ("MFA bypass").
* **Résolution :** Ajout du paramètre obligatoire `-FederatedIdpMfaBehavior "acceptIfMfaDoneByFederatedIdp"` à la commande. Cela indique à Microsoft : "Fais confiance à Keycloak, le MFA a déjà été validé de leur côté."
* **Résultat :** La commande retourne le tableau de configuration. Le domaine est officiellement fédéré.

---

## Étape 4 : Tests d'Enrôlement Windows (Entra Join) et Debugging de l'URL

### Sous-étape 4.1 : L'absence de l'option "Other User"
* **Problème :** Sur l'écran de verrouillage Windows, impossible de se connecter avec un utilisateur du domaine.
* **Diagnostic :** La commande `dsregcmd /status` affichait `AzureAdJoined : NO`.
* **Explication :** La machine n'était qu'un poste de travail local isolé (Workgroup). Elle devait d'abord être jointe au domaine Entra ID ("Access work or school" > Connect > "Join this device to Microsoft Entra ID").

### Sous-étape 4.2 : L'Erreur 404 (Event ID 360)
* **Problème au moment du Join :** Après avoir entré l'email de l'utilisateur, une fenêtre blanche apparaît avec le message générique : `"Something went wrong"`.
* **Diagnostic Avancé (Event Viewer) :**
  * Chemin : `Applications and Services Logs > Microsoft > Windows > User Device Registration > Admin`.
  * L'Event ID 360 indiquait un échec du provisionnement.
* **Test de l'URL via Microsoft Edge :** En collant l'URL `$logOnUri` configurée dans le script PowerShell (`https://auth.charif-labs.tech/application/saml/entra-saml-provider/sso/binding/redirect/`) directement dans le navigateur du VM, Keycloak a retourné une page **"Not Found"**.
* **Explication technique :** L'URL de redirection SAML d'Keycloak est générée dynamiquement en fonction du champ **Slug** de l'Application, et non du nom du Provider. L'application créée dans Keycloak avait un slug différent de `entra-saml-provider`. Microsoft tentait donc de contacter une page inexistante.
* **Résolution :** Au lieu de relancer un script PowerShell complexe pour modifier la configuration côté Microsoft, la solution la plus rapide a été de modifier le "Slug" de l'application directement dans l'interface d'Keycloak pour qu'il soit exactement `entra-saml-provider`. Le pop-up de connexion a fonctionné immédiatement après.

---

## Étape 5 : L'Alignement Granulaire des Identités (La condition finale)
* **Le Problème latent :** Lors du test de connexion sur la mire Keycloak (via le pop-up Entra), des erreurs de mot de passe ou des échecs de validation peuvent survenir si les profils utilisateurs ne sont pas synchronisés.
* **L'Exigence Technique (Le Contrat SAML) :**
  Pour qu'Entra ID valide la connexion Windows, la chaîne d'identification doit être mathématiquement parfaite entre les deux systèmes :
  1. Le **User Principal Name (UPN)** dans Entra ID (ex: `user-basic-01@ms.charif-labs.tech`).
  2. Doit correspondre à 100% au **NameID** envoyé par l'assertion SAML d'Keycloak.
  3. Doit correspondre à l'**Email** configuré dans l'onglet Directory > Users d'Keycloak.
* **Action requise :** Auditer chaque utilisateur test pour s'assurer que leur adresse email dans Keycloak est identique à leur UPN dans Entra ID, et que le *NameID Property Mapping* du SAML Provider pointe bien vers cet attribut.