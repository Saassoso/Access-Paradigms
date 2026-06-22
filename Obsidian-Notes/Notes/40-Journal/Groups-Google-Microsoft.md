

### Step 1: Create the Groups in Google Workspace

Google identifies groups by their **email address**.

1. Log in to the [Google Workspace Admin Console](https://admin.google.com).
2. On the left menu, go to **Directory** > **Groups**.
3. Click **Create group** at the top.
4. Fill in the details for your first group:
* **Name:** `IT`
* **Group email:** `it@charif-labs.tech`
1. Set the Access settings (usually "Restricted" is fine for internal IAM groups) and click **Create group**.



### Step 2: Create the Groups in Microsoft Entra ID

Microsoft identifies groups by an alphanumeric **Object ID**.

1. Log in to the [Microsoft Entra Admin Center](https://www.google.com/search?q=https://entra.microsoft.com/).
2. On the left menu, expand **Identity**, click **Groups**, then select **All groups**.
3. Click **New group** at the top.
4. Fill in the group details:
* **Group type:** Select `Security`.
* **Group name:** `IT`
* **Membership type:** `Assigned`.
	* Click **Create** at the bottom.
	* Repeat this for **Developers** and **Comptabilite**.

Once created, click on each group in the list to open its Overview page.
![](images/Groups%20in%20Google%20and%20Microsoft%20environments-2.png)
![](images/Groups%20in%20Google%20and%20Microsoft%20environments-1.png)
#### Object-ID
- **IT** : 4660cc7a-aa46-494a-a462-327f7e168fc0
- **Developers** : cf56b3ec-647a-495f-9d3c-62531d110892
- **Comptabilite** : 0c3b5616-2c2b-48b9-b9a4-f2845a6a5b29

Wuqa658727@
### Step 3: Get the UUIDs from Keycloak

Keycloak also identifies groups by an alphanumeric **UUID**.
1. Log in to your Keycloak Admin Console.
2. Go to **Groups** on the left menu.
3. Expand your `Bureau 2` group and click on the **IT** sub-group.
Look at group Id in your Browser URL :
- **IT** : 66216f82-27c0-4f65-b4fe-e64941f1c05f/ebad6de7-1f9f-43f9-8cf0-a7dd8dcaeb08
- **Developers** : 66216f82-27c0-4f65-b4fe-e64941f1c05f/f5f55461-1fe9-409e-9e8a-f5c8855dabf5
- **Comptabilite** : 66216f82-27c0-4f65-b4fe-e64941f1c05f/01926d65-ed6b-4537-ae3c-7a82081d2747

---

### Step 4: Wire it all together in n8n

Now that you have your master list of IDs, it is time to plug them into n8n so it can translate Keycloak's language into Google's and Microsoft's languages.

1. Open your n8n workflow.
2. Open the **`Translate Group ID (Add)`** Set node.
3. Under the `targetGoogleGroup` value, replace the placeholder logic with your actual Keycloak UUIDs and Google Emails. It should look like this:
```javascript
{{ $json.groupId === 'paste-keycloak-it-uuid-here' ? 'it@charif-labs.tech' : $json.groupId === 'paste-keycloak-dev-uuid-here' ? 'developers@charif-labs.tech' : $json.groupId === 'paste-keycloak-comptabilite-uuid-here' ? 'comptabilite@charif-labs.tech' : null }}

```


4. Under the `targetEntraGroupId` value, do the exact same thing, but mapping the Keycloak UUIDs to the Entra Object IDs:
```javascript
{{ $json.groupId === 'paste-keycloak-it-uuid-here' ? 'paste-entra-it-object-id-here' : $json.groupId === 'paste-keycloak-dev-uuid-here' ? 'paste-entra-dev-object-id-here' : $json.groupId === 'paste-keycloak-comptabilite-uuid-here' ? 'paste-entra-comptabilite-object-id-here' : null }}

```


5. **CRITICAL:** Copy those exact same formulas and paste them into your **`Translate Group ID (Remove)`** Set node so that the logic works for deleting users from groups too.

Once those IDs are pasted in, your workflow is fully locked and loaded. Are you ready to trigger a real test event from Keycloak?