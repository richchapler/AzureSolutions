# Surface Data: Cognitive Search, RBAC-secured PowerApp (WiP)
_Note: Cognitive Search is also known as "Search Services" and "Azure Search"_

<img src="https://user-images.githubusercontent.com/44923999/231556765-3276ee63-1883-4176-af10-36a1ea3a9e12.png" width="1000" />

## Use Case
This solution considers the following requirements:

* "We want to share our secured Cognitive Search index with our community of business users"
* "We want to use Role-Based Access Control {i.e., Azure Active Directory} for all security"
* "Since the Demo App only supports QueryKey-based security, we need an alternate solution"

## Prerequisites
This solution requires the following resources:

* [**Application Registration**](Infrastructure_ApplicationRegistration.md)
* [**Cognitive Search**](https://azure.microsoft.com/en-us/products/search)
* [**Power Apps**](https://powerapps.microsoft.com/en-us/)
* [**Power Automate**](https://powerautomate.microsoft.com/en-us/)

## Proposed Solution
This solution will address requirements in three exercises:

* Exercise 1: Create Index
* Exercise 2: Secure Index
* Exercise 3: Create Flow
* Exercise 4: Create App

-----

## Exercise 1: Create Index
In this exercise, we will create an index using sample data.

### Step 1: Create Index

Navigate to Cognitive Search, "**Overview**" and then click "**Import data**".

<img src="https://user-images.githubusercontent.com/44923999/230911383-a7135ef7-65e2-45e7-bb4f-38f5addaa8e9.png" width="800" title="Snipped: April 10, 2023" />

On the "**Connect your data**" tab, select "**Samples**" using the "**Data Source**" drop-down menu.<br>
Select "**realestate-us-sample**" from the resulting data source list and then, click "**Next: Add cognitive skills (Optional)**".

<img src="https://user-images.githubusercontent.com/44923999/230911764-3ed44b43-4673-416b-9882-87b809e957cd.png" width="800" title="Snipped: April 10, 2023" />

On the "**Add cognitive skills (Optional)**" tab, simply click "**Skip to: Customize target index**".

<img src="https://user-images.githubusercontent.com/44923999/230911991-df3cabf9-8f28-48f1-95b7-41cb9c1458d5.png" width="800" title="Snipped: April 10, 2023" />

On the "**Customize target index**" tab, review default configuration, modify if desired, and then click "**Next: Create an indexer**".

<img src="https://user-images.githubusercontent.com/44923999/230913029-a23d514c-5fb8-40e3-937c-7d1757b031a0.png" width="800" title="Snipped: April 10, 2023" />

On the "**Create an indexer**" tab, review default configuration, modify if desired, and then click "**Submit**".

<img src="https://user-images.githubusercontent.com/44923999/230914454-a2730070-4811-40f1-8005-4530a30bf561.png" width="800" title="Snipped: April 10, 2023" />

Allow time for processing and then click into the new "**realestate-us-sample-index**" index.

<img src="https://user-images.githubusercontent.com/44923999/230914830-9de5c8fe-8475-44d1-9b05-47ad258e6325.png" width="800" title="Snipped: April 10, 2023" />

On the "**Search explorer**" tab, click "**Search**" and review results.

### Step 2: Create Demo App

Click "**Create Demo App**".

<img src="https://user-images.githubusercontent.com/44923999/230915524-487c5670-165b-4487-89e5-0040c41210cf.png" width="800" title="Snipped: April 10, 2023" />

On the resulting "**Create Demo App**" pop-up, click "**Enable CORS and continue**".

<img src="https://user-images.githubusercontent.com/44923999/230916396-cdbc6bbf-5cd2-4a90-84ff-e6f1e307d704.png" width="800" title="Snipped: April 10, 2023" />

On the "**Individual result**" tab, review default configuration, modify if desired, and then click "**Create Demo App**".
_Note: Review "Sidebar" and "Suggestions" tab, as desired._

<img src="https://user-images.githubusercontent.com/44923999/230918267-a9df50e2-21f9-40ae-8d82-22293a2db82c.png" width="800" title="Snipped: April 10, 2023" />

On the resulting "**Your demo app is ready**" pop-up, click "**Download**".<br>
Use Windows Explorer to open your "**Downloads**" folder and then the new "**AzSearch.html**" file.

<img src="https://user-images.githubusercontent.com/44923999/230919150-61603545-44b7-41c9-8066-a1150bc9f455.png" width="800" title="Snipped: April 10, 2023" />

## Exercise 1: Secure Index
In this exercise, we will configure index security.

The default Demo App {i.e., **AzSearch.html**} includes the index "**queryKey**" directly in the auto-generated HTML:

```
var automagic = new AzSearch.Automagic({ index: "{INDEX_NAME}", queryKey: "{QUERY_KEY}", service: "{SEARCH_SERVICE_NAME}", dnsSuffix:"search.windows.net" });
```

If we remove the queryKey, the Demo App will not work... and, `AzSearch.Automagic(...` only works with queryKey.<br>

To secure the index using Role-Based Access Control (RBAC), we must:

* Modify the "**API Access Control**" setting
* Programmatically set index permissions for an Azure Active Directory user or group

### Step 1: Modify "**API Access Control**"

Navigate to Cognitive Search, and then "**Keys**" in the "**Settings**" grouping of the left-hand navigation.

<img src="https://user-images.githubusercontent.com/44923999/230929406-ae8bc92b-109b-48ff-9932-40202c027840.png" width="800" title="Snipped: April 10, 2023" />

Under the "**API access control**" header, click to select the "**Role-based access control**" radio button.

### Step 2: Programmatically set index permissions

Navigate to the **Cloud Shell**, configure as required, and select **Powershell**.

<img src="https://user-images.githubusercontent.com/44923999/230935664-52785fd8-e396-4bd9-b8ca-75fd75dd1467.png" width="800" title="Snipped: April 10, 2023" />

Modify, copy / paste, and then run the following command:

``` PowerShell
New-AzRoleAssignment -ObjectId {OBJECT_ID}
  -RoleDefinitionName "Search Index Data Reader"
  -Scope "/subscriptions/{SUBSCRIPTION_ID}/resourceGroups/{RESOURCE_GROUP}/providers/Microsoft.Search/searchServices/{SERVICE_NAME}/indexes/{INDEX_NAME}"
```
Logic Explained:

* `{OBJECT_ID}` ... refers to the Object ID for the Application Registration

_Note: It is not possible to create a "Search Index Data Reader" role assignment for a Service Principal._

You can expect a result like:

```
RoleAssignmentName : 89f76855-6dbc-470b-aae2-177e7a203c27
RoleAssignmentId   : /subscriptions/{SUBSCRIPTION_ID}/resourceGroups/{RESOURCE_GROUP}/providers/Microsoft.Search/searchServices/{SERVICE_NAME}/indexes/{INDEX_NAME}/providers/Microsoft.Authorization/roleAssignments/89f76855-6dbc-470b-aae2-177e7a203c27
Scope              : /subscriptions/{SUBSCRIPTION_ID}/resourceGroups/{RESOURCE_GROUP}/providers/Microsoft.Search/searchServices/{SERVICE_NAME}/indexes/{INDEX_NAME}
DisplayName        : {USER_NAME}
SignInName         : {USER_SIGNIN}
RoleDefinitionName : Search Index Data Reader
RoleDefinitionId   : 1407120a-92aa-4202-b7e9-c0e197c71c8f
ObjectId           : {OBJECT_ID}
ObjectType         : User
CanDelegate        : False
Description        : 
ConditionVersion   : 
Condition          : 
```

#### Step 3c: Confirm Success

Navigate to the User in Azure Active Directory, and then click "**Azure role assignments**" in the left-hand navigation.

<img src="https://user-images.githubusercontent.com/44923999/230096988-81b6e89a-8715-4719-be15-5e0c4e31461b.png" width="800" title="Snipped: April 10, 2023" />

-----

**Congratulations... you have successfully completed this exercise**

-----

## Exercise 3: Create Flow
In this exercise, we will create a Power Automate Flow that calls the Cognitive Search API.

<img src="https://user-images.githubusercontent.com/44923999/231559914-ca3c7db4-68d8-4464-8be0-8bede8546dfc.png" width="800" title="Snipped: April 12, 2023" />

Navigate to Power Automate, sign in if required, click "**+ Create**", and then click "**Instant cloud flow**" on the resulting page.

<img src="https://user-images.githubusercontent.com/44923999/231560981-e54f2e71-ea32-454c-b314-d637a2aa03d3.png" width="800" title="Snipped: April 12, 2023" />

On the resulting "**Build**..." pop-up, enter a meaningful "**Flow name**" and select "**PowerApps (V2)**" from the "**Choose**..." list.<br>
Click "**Create**".

<img src="https://user-images.githubusercontent.com/44923999/231562018-02edea1a-3fa3-4a7b-86de-eb685d170b44.png" width="800" title="Snipped: April 12, 2023" />

On the new flow designer page, click to expand the "**PowerApps (V2)**" trigger, and then click "**+ Add an input**".

<img src="https://user-images.githubusercontent.com/44923999/231563868-e1934336-6fe9-4152-9d8f-07fbef593e94.png" width="800" title="Snipped: April 12, 2023" />

Select "**Text**" from the resulting "**Choose the type of user input**" list.

<img src="https://user-images.githubusercontent.com/44923999/231762469-b2dfca21-6c56-47a6-b4a7-01fbdb6a6298.png" width="800" title="Snipped: April 13, 2023" />

Prompt | Entry
:----- | :-----
**Input** | Enter "**Query string**"
**Description** | Enter "**Examples: *, $top=10, $top=10&search=***"

_Note: These entries match the Search Explorer._

Click "**Save**" and then "**+ New Step**".

<img src="https://user-images.githubusercontent.com/44923999/231763351-47f03570-3b94-4206-9d34-b45bdf9ea76d.png" width="800" title="Snipped: April 13, 2023" />

On the new "**Choose an operation**" step, search for and select "**HTTP**".

<img src="https://user-images.githubusercontent.com/44923999/231765436-6163a17f-8687-4c9e-8e07-a45b2be52cdb.png" width="800" title="Snipped: April 13, 2023" />

Prompt | Entry
:----- | :-----
**Method** | Select "**GET**"
**URI** | Modify and enter:<br>`https://{SERVICE_NAME}.search.windows.net/indexes/{INDEX_NAME}/docs?api-version=2021-04-30-Preview&search=`<br>Click "**Add dynamic content**" and then select "**Query string**" to complete the entry
**Headers** |Modify and enter key-value pair: `api-key` :: `{QUERY_KEY}`

Click "**Save**" and then "**+ New Step**".

<img src="https://user-images.githubusercontent.com/44923999/231766720-007b56fc-36ac-4fc0-90e8-c4c2df744f0b.png" width="800" title="Snipped: April 13, 2023" />

On the new "**Choose an operation**" step, search for and select "**Respond to a PowerApp or flow**".

<img src="https://user-images.githubusercontent.com/44923999/231766835-1264402d-3de0-4a72-9897-5aa3e0be58f0.png" width="800" title="Snipped: April 13, 2023" />

On the new "**Respond to a PowerApp or flow**" step,  click "**+ Add an output**".

<img src="https://user-images.githubusercontent.com/44923999/231767272-3f08c7ec-1463-462c-b50f-4cab3e4c040d.png" width="800" title="Snipped: April 13, 2023" />

Select "**Text**" from the resulting "**Choose the type of output**" list.

<img src="https://user-images.githubusercontent.com/44923999/231767570-8d7de38e-9a88-422e-a500-680c5c98cfd3.png" width="800" title="Snipped: April 13, 2023" />

Prompt | Entry
:----- | :-----
**Enter title** | Enter "**Query string**"
**Enter a value to respond** | Click "**Add dynamic content**" and then select "**Body**" from the "**HTTP**" grouping

Click "**Save**".

### Confirm Success

Click "Test".

<img src="https://user-images.githubusercontent.com/44923999/231768843-7cea7b3b-44dc-465b-a2cb-a2388270489c.png" width="800" title="Snipped: April 13, 2023" />

In the "**Test Flow**" pop-out, click the radio button for "**Manually**" and then click "**Test**".

<img src="https://user-images.githubusercontent.com/44923999/231769672-d4851498-c2c5-4ed0-be04-39d642001f14.png" width="800" title="Snipped: April 13, 2023" />

In the "**Run Flow**" pop-out, enter a "**Query string**" {e.g., `$top=1`} and then click "**Run flow**".

<img src="https://user-images.githubusercontent.com/44923999/231769944-55ee3df6-8d8e-48ea-881e-2bdbecb6aca8.png" width="800" title="Snipped: April 13, 2023" />

Click "Done".

<img src="https://user-images.githubusercontent.com/44923999/231770153-27803ff6-1441-4d09-861b-e4091704c311.png" width="800" title="Snipped: April 13, 2023" />

You can expect a "**Your flow ran successfully**" message, along with a clickable visualization of the results.

_NOTE: OPEN AZURE SUPPORT CASE SUPPORTING USE OF RBAC FOR API CALL INSTEAD OF QUERY KEY_

-----

**Congratulations... you have successfully completed this exercise**

-----

## Exercise 4: Create App
In this exercise, we will create a Power App that queries "**realestate-us-sample-index**" (via Power Automate >> API request).

<img src="https://user-images.githubusercontent.com/44923999/231806782-153f19b6-f8a7-4ed8-95fd-9fc00b2c32e3.png" width="800" title="Snipped: April 13, 2023" />

Navigate to Power Apps, sign in if required, click "**+ Create**", and then click "**Blank app**" on the resulting page.

<img src="https://user-images.githubusercontent.com/44923999/231807007-db6ddbfd-7c80-46e3-9976-5919695d9458.png" width="800" title="Snipped: April 13, 2023" />

On the resulting pop-up, click "**Create**" in the "**Blank canvas app**" rectangle.

<img src="https://user-images.githubusercontent.com/44923999/231807447-0a4b5d10-998c-41ce-9b9f-5bdfdb8497cc.png" width="800" title="Snipped: April 13, 2023" />

On the resulting "Canvas app from blank" pop-up, enter a meaningful "**App name**" name and select format "**Tablet**".<br>
Click "**Create**".

<img src="https://user-images.githubusercontent.com/44923999/231812219-9bd174c6-ddf6-4ec3-9d4a-a3aece0b9c2c.png" width="800" title="Snipped: April 13, 2023" />

### Insert Controls
Click "**Add an item from the Insert pane**".

<img src="https://user-images.githubusercontent.com/44923999/231812785-0b6bd99f-d782-48b5-8e6e-e773efe2d9f0.png" width="800" title="Snipped: April 13, 2023" />

Insert the following controls:
*	**Text Input** … for search query input
*	**Button** … for search query submit
*	**Text Label** … for search result display

<img src="https://user-images.githubusercontent.com/44923999/231859529-27ce90f8-704a-4c77-92a9-d4ad59750b8e.png" width="800" title="Snipped: April 13, 2023" />

Consider personalizing look-and-feel {e.g., control width, border size, color, etc.} to enhance usability; examples:
*	**Text Input** … 1) delete "Default" to "No value", and 2) change "Hint Text" to "Examples: *, $top=10, $top=10&search=*"
*	**Button** … 1) change "Text" property to value "Search"
*	**Text Label** … 1) change "Border thickness" to "1" and 2) change "Vertical align" to "Top"

### Enable Power Automate

Click the cog icon in the upper right of the screen to open the Settings pop-up menu.

<img src="https://user-images.githubusercontent.com/44923999/231860561-393dc7f0-6eda-4d07-b651-3318bed1ba72.png" width="800" title="Snipped: April 13, 2023" />

Navigate to "**Upcoming Features**", and then the "**Retired**" tab.<br>
Scroll to the bottom of the options and switch "**Enable classic Power Automate pane**" to "**On**".<br>
Close the pop-up menu.

<img src="https://user-images.githubusercontent.com/44923999/231861437-d112438f-3c84-48c6-863e-8fd318a8376b.png" width="800" title="Snipped: April 13, 2023" />

Click the ellipses on the right end of the menu bar and then select "Power Automate" from the resulting drop-down menu.

<img src="https://user-images.githubusercontent.com/44923999/231862323-15b068e2-7ffb-4fa2-85a7-b585a4ede895.png" width="800" title="Snipped: April 13, 2023" />

Select the previously-created Power Automate Flow and then close the pop-up menu.<br>
Click on the red circle-X icon and then select "**Edit in the formula bar**" from the drop-down menu.<br>
Delete the contents of the formula bar.

### Change Button OnSelect

Click on the button control and then the "**Advanced**" tab in the resulting pop-out menu.

<img src="https://user-images.githubusercontent.com/44923999/231863997-8bc340ce-3031-4035-a83d-d4577d6048c9.png" width="800" title="Snipped: April 13, 2023" />

Populate the "OnSelect" textbox with: `Set(searchAPI,rchaplericf.Run(TextInput1.Text));`

Logic Explained:

* `Set(searchAPI...` ... will take the value returned by Power Automate and assign it to variable `searchAPI`
* `rchaplericf.Run(TextInput1.Text)` ... will call the Power Automate, Instant Cloud Flow (which I named `rchaplericf`) and run it with the query string entered in the Text Input control

### Surface Results

Click on the text label control and then the "**Advanced**" tab in the resulting pop-out menu.

<img src="https://user-images.githubusercontent.com/44923999/231866008-ac2fca29-1deb-4b62-aff2-8ed4a601b93c.png" width="800" title="Snipped: April 13, 2023" />

Populate the "Text" textbox with: `searchAPI.results`<br>
Click "**Save**".

### Confirm Success



Click the play icon {i.e., "Preview the app (F5)"} in the upper-right of the menu-bar.

<img src="https://user-images.githubusercontent.com/44923999/231867004-323f9426-1495-46a2-903d-725c150ba8a6.png" width="800" title="Snipped: April 13, 2023" />

Enter a query string and then click "**Search**".

TO-DO: JSON PARSING AND GALLERY PRESENTATION

-----

**Congratulations... you have successfully completed this exercise**

-----

## Why I didn't use API

* Search API Authorization is possible only for API Keys (not allowed by use case) or Bearer Tokens (OAuth)
* OAuth uses Application Registration credentials {i.e., Client ID and Client Secret} to generate a Bearer Token
  _You cannot request a bearer token from Azure Active Directory (AAD) OAuth without a Client_ID. The OAuth 2.0 client credentials grant flow permits a web service (confidential client) to use its own credentials, instead of impersonating a user, to authenticate when calling another web service_
* Application Registrations can be assigned a role for a Search Service, but not a Search Index 
* Even if we were able to authenticate via delegation {i.e., use user permissions}, the authorized user would still have access to everything in the Search Service because the authorization would be based on the Application Registration
* SDK sits on top of the API so has the same limitations

### Ideas
* EXPLORE WHETHER POWER AUTOMATE HAS A KEY VAULT CONNECTOR THAT WOULD USE CURRENT USER CREDENTIALS TO SEE IF THEY HAVE ACCESS TO THE QUERY KEY
* LOCK DOWN POWER APP TO ONLY USERS THAT SHOULD HAVE ACCESS AND HAVE ONLY A SINGLE INDEX IN THE SEARCH SERVICE

-----

## Reference

* [Azure Cognitive Search Service REST](https://learn.microsoft.com/en-us/rest/api/searchservice/)
* [Connect to Azure Cognitive Search using Azure role-based access control (Azure RBAC)](https://learn.microsoft.com/en-us/azure/search/search-security-rbac?tabs=config-svc-rest%2Croles-portal%2Ctest-portal%2Ccustom-role-portal%2Cdisable-keys-portal)
* [Authorize access to a search app using Azure Active Directory](https://learn.microsoft.com/en-us/azure/search/search-howto-aad?tabs=config-svc-portal%2Caad-rest)
* [OAuth 2.0 authentication with Azure Active Directory](https://learn.microsoft.com/en-us/azure/active-directory/fundamentals/auth-oauth2)
* [Microsoft identity platform and the OAuth 2.0 client credentials flow](https://learn.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow)