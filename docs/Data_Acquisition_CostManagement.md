# Data Acquisition: Azure Cost Management API

![image](https://user-images.githubusercontent.com/44923999/188199195-34c228d5-37e8-4c06-8d7d-88b0e8d2a3ec.png)

This documentation consolidates previous posts on this topics covering the acquisition of data using the Azure Cost Management API via: 1) Data Factory, 2) Logic App, and 3) Function App (**CREATE LINKS FOR EACH OF THESE SECTIONS**)

For any of these resource types, your answer to "why use X?" may be as simple as the fact that you favor that solution type

| Resource Type | Pros                                                         | Cons                                                         |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Data Factory  |                                                              | - Nested iteration {i.e., `foreach` inside of a `foreach`} is not possible |
| Logic App     | - No-Code interface might be more accessible for some developers | - Interface is challenging<br />- Processing is slow (and it is not possible to see what is happening until all iteration is complete) |
| Function      | - Ability to leverage C# is accessible to more advanced developers |                                                              |

### Step 1a: Prepare Infrastructure

All solutions require the following resources:

* [**Application Registration**](Infrastructure_ApplicationRegistration.md) with "Cost Management Reader" role assignment (granted at Subscription or Resource Group)
* Data Explorer [**Cluster**](Infrastructure_DataExplorer_Cluster.md) and [**Database**](Infrastructure_DataExplorer_Database.md)

Solution-specific requirements will be included in each "...via" section

#### Prepare Data Explorer

* Navigate to Data Explorer Database, then Query in the Data grouping of the left-hand navigation

* Run the following KQL:

  ```KQL
  .create table CostManagement (
      PreTaxCost: decimal
      , UsageDate: int
      , ResourceGroupName: string
      , ResourceType: string
      , ResourceId: string
      , ResourceLocation: string
      , MeterCategory: string
      , MeterSubCategory: string
      , Meter: string
      , ServiceName: string
      , PartNumber: string
      , PricingModel: string
      , ChargeType: string
      , ReservationName: string
      , Frequency: string
      , Currency: string
      )
  ```



## ...via Data Factory

This use case considers requirement statements like:

* "We want to analyze Azure resource costs across many subscriptions"
* "An understanding of resources and costs would really help us get a handle on inherited subscriptions"
* "Power BI, Cost Management connector requires billing account permissions we cannot get"

### Step 1b: Prepare Infrastructure

This solution requires the following resources:

* [Synapse](Infrastructure_Synapse.md) with a Data Explorer [Integration Dataset](Infrastructure_Synapse_Dataset.md) and Data Explorer, "**AllDatabasesAdmin**" permissions for the Synapse, System-Assigned Managed Identity

### Step 2: Create Destination Table
In this step, we will create the Data Explorer table which will serve as destination for our Synapse Pipeline

* Navigate to Data Explorer and then select Query from the navigation

  <img src="https://user-images.githubusercontent.com/44923999/188202710-018bd814-90f5-4283-a70c-29904f47512e.png" width="800" title="Snipped: September 2, 2022" />

* **Run** the following KQL:

  ```
  .create table CostManagement (
      Currency: string
      , PreTaxCost: real
      , ResourceType: string
      , UsageDate: string
      )
  ```

_Note: This initial version of the table includes only a few of the columns that might be included as you evolve your query logic_

### Step 3: Create Linked Service
In this step, we will create the Linked Service we will use to get our Bearer Token.

* Open **Synapse Studio** and click the **Manage** navigation icon
* Click "**Linked services**" from the "**External connections**" grouping in the resulting navigation
* Click "**+ New**"

  <img src="https://user-images.githubusercontent.com/44923999/188169292-9886c049-e9cb-4d11-b6d9-a7ffe7e285d9.png" width="800" title="Snipped: September 2, 2022" />

* Search for and select "REST" on the "**New linked service**" pop-out and then click **Continue**

  <img src="https://user-images.githubusercontent.com/44923999/188169910-55df4aad-6c9f-434a-8fe9-f0b321aee9af.png" width="800" title="Snipped: September 2, 2022" />

* Complete the resulting "**New linked service**" pop-out, including:

  | Prompt                  | Entry                                                        |
  | ----------------------- | ------------------------------------------------------------ |
  | **Base URL**            | Modify and enter:<br>`https://management.azure.com/subscriptions/{Subscription Id}/providers/Microsoft.CostManagement/query?api-version=2021-10-01` |
  | **Authentication Type** | Select **Anonymous**                                         |

* Click "**Test connection**" to confirm successful connection and then click **Create**

_Note: Consider including the Subscription Name in your Linked Service and Dataset names_

### Step 4: Create Dataset
In this step, we will create the Dataset we will use to get our Bearer Token.

* Open **Synapse Studio** and click the **Data** navigation icon
* Click **+** and then select "**Integration dataset**" from the **Linked** grouping in the resulting navigation

  <img src="https://user-images.githubusercontent.com/44923999/188195944-563f1dae-23eb-487d-b2d0-1747a6058ee1.png" width="800" title="Snipped: September 2, 2022" />

* Search for and select **REST** on the "**New linked service**" pop-out and then click **Continue**

  <img src="https://user-images.githubusercontent.com/44923999/188196140-3ac10408-7991-4702-8d64-568f74b281c5.png" width="800" title="Snipped: September 2, 2022" />

* Complete the "**Set properties**" pop-out form and then click **OK**
* Click "**Test connection**" to confirm successful connection
* Click "**Publish all**", review changes on the resulting "**Publish all**" pop-out and then click **Publish**

### Step 5: Create Pipeline
In this step, we will create a Pipeline and add Activities.

* Open "**Synapse Studio**" and click the **Integrate** navigation icon
* Click **+** and select **Pipeline** from the resulting dropdown menu

### Step 5a: Create Activity 1, Get Token
This activity will make a REST API call and get a bearer token.

* Expand **General** in the **Activities** bar
* Drag-and-drop a **Web** component into the main window
* Complete the form on the **Settings** tab

  <img src="https://user-images.githubusercontent.com/44923999/188197200-3f5083fd-fcc3-4a92-bbfa-bde3e01e77dc.png" width="800" title="Snipped: September 2, 2022" />

  | Prompt      | Entry                                                        |
  | ----------- | ------------------------------------------------------------ |
  | **URL**     | Modify and enter:`https://login.microsoftonline.com/{TenantId}/oauth2/token` |
  | **Method**  | Select **POST**                                              |
  | **Headers** | Click "**+ Add**" and enter key-value pair: `content-type` :: `application/x-www-form-urlencoded` |
  | **Body**    | Modify and enter:<br> `grant_type=client_credentials&client_id={Client Identifier}&client_secret={Client Secret}& resource=https://api.loganalytics.io/` |

* Click **Debug** and confirm success

  <img src="https://user-images.githubusercontent.com/44923999/188210464-6d5a3462-9777-4523-a29c-9b239592ad60.png" width="800" title="Snipped: September 2, 2022" />

* On the resulting **Output**, mouse over the "**Get Token**" row and click on the **Output** icon

  <img src="https://user-images.githubusercontent.com/44923999/188210549-725e73ff-5c3b-4890-90ec-6cbf87b7e25d.png" width="800" title="Snipped: September 2, 2022" />

* Copy the "access token" value for later use... in the example below, everything from `eyJ0...` to `...1mqg`

  ```
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsIng1dCI6IjJaUXBKM1VwYmpBWVhZR2FYRUpsOGxWMFRPSSIsImtpZCI6IjJaUXBKM1VwYmpBWVhZR2FYRUpsOGxWMFRPSSJ9.eyJhdWQiOiJodHRwczovL21hbmFnZW1lbnQuYXp1cmUuY29tLyIsImlzcyI6Imh0dHBzOi8vc3RzLndpbmRvd3MubmV0LzE2YjNjMDEzLWQzMDAtNDY4ZC1hYzY0LTdlZGEwODIwYjZkMy8iLCJpYXQiOjE2NjIxMzU3MzgsIm5iZiI6MTY2MjEzNTczOCwiZXhwIjoxNjYyMTM5NjM4LCJhaW8iOiJFMlpnWURnZXE4L2x6ZGlTWmFNdmNQTDVtUXZyQUE9PSIsImFwcGlkIjoiNzVhZmM4ZTktZjI5Ny00YmE0LThiNWItNWNlMzQ5NTI1OGExIiwiYXBwaWRhY3IiOiIxIiwiaWRwIjoiaHR0cHM6Ly9zdHMud2luZG93cy5uZXQvMTZiM2MwMTMtZDMwMC00NjhkLWFjNjQtN2VkYTA4MjBiNmQzLyIsImlkdHlwIjoiYXBwIiwib2lkIjoiZmY1Njc1NzQtMzY2Ny00ZDJiLWIxNmUtMmMyOTJjOTZkYzBkIiwicmgiOiIwLkFVWUFFOEN6RmdEVGpVYXNaSDdhQ0NDMjAwWklmM2tBdXRkUHVrUGF3ZmoyTUJPQUFBQS4iLCJzdWIiOiJmZjU2NzU3NC0zNjY3LTRkMmItYjE2ZS0yYzI5MmM5NmRjMGQiLCJ0aWQiOiIxNmIzYzAxMy1kMzAwLTQ2OGQtYWM2NC03ZWRhMDgyMGI2ZDMiLCJ1dGkiOiJ3QklycG9nYzIwZThxdXZ0NjhLT0FBIiwidmVyIjoiMS4wIiwieG1zX3RjZHQiOjE2NDUxMzcyMjh9.JSSyRQzqkcDatL9d8Vlbd8pr6_FXPmYJdf5fiZI1zWbQalH48pIVF57l4ZiWEwQLGdzsfZEwZQN9-ujSUWPSXGIUw3iTGqtAl8sI48G4hD8rPB9YwCt8sSKWVd1vx30_f7Cm4CyDiY7qNqkOfNdADzfpBj-CwB2U4zhmrVzbFk57qIpAuXUDDlAVen6JKXokC931mmUpZ_fYDBHiE14tXgPalPklP2bxaJKTlahObvfuLworr-A70zJi2Pdp5ckmMR3GqySM34Hz9uMbKtnEs8fJNJRYaeVGSHIjbY0Zp0wXgwmcaK31TXsNOl1_HXoMwvwLPicqR1y28Mq0ef1mqg"
  ```

### Step 5b: Create Activity 2, Copy Data
This activity will make a REST API call, parse the response and write the data to Data Explorer.

* Expand "**Move & Transform**" in the **Activities** bar
* Drag-and-drop a "**Copy Data**" component into the activity window
* Create a dependency from the "**Get Token**" component to the "**Copy Data**" component

  <img src="https://user-images.githubusercontent.com/44923999/188206412-e90cef93-615e-403f-8e66-64d46fb9af86.png" width="800" title="Snipped: September 2, 2022" />

#### "Source" tab

* Enter values on the **Source** tab 

  | Prompt                 | Entry                                                        |
  | ---------------------- | ------------------------------------------------------------ |
  | **Source dataset**     | Select your REST dataset                                     |
  | **Request method**     | Select **POST**                                              |
  | **Additional headers** | Click **+ Add** and enter key-value pairs:<br>`content-type` :: `application/json;charset=utf-8`<br>`authorization` :: `@concat('Bearer ',activity('Get Token').output.access_token)` |

  Finally, paste the following JSON into '**Request body**':

  ```
  {
        "type": "Usage",
        "timeframe": "Custom",
        "timePeriod": {
            "from": "@{formatDateTime(adddays(utcnow(),-2),'yyyy-MM-ddT00:00:00')}",
            "to": "@{formatDateTime(utcnow(),'yyyy-MM-ddTHH:mm:ss')}"
        },
        "dataset": {
            "granularity": "Daily",
            "aggregation": {
                "totalCost": {
                    "name": "PreTaxCost",
                    "function": "Sum"
                }
            },
            "grouping": [
                {
                    "type": "Dimension",
                    "name": "ResourceType"
                }
            ]
        }
    }
  ```
  
  _Notes:_
  * _Time period values {e.g., 2022-01-01T00:00:00} are parameterized for use in a dynamic expression in the example above_
  * _The API limits how many dimensions can be included to 15_
  * _See links in Reference (below) for additional help with preparation of Request Body JSON {i.e., the query}_
  
* Click "Preview data"  

  <img src="https://user-images.githubusercontent.com/44923999/188211809-54c79983-ef20-47ef-b754-4909164d1cf5.png" width="800" title="Snipped: September 2, 2022" />

* On the resulting "**Please provide actual value of the parameters to preview data**" pop-out, paste the previously-copied Bearer Token value into the **Value** textbox and then click **OK** 

  <img src="https://user-images.githubusercontent.com/44923999/188220207-9c8c7891-868a-4d99-883d-bada3de2996c.png" width="800" title="Snipped: September 2, 2022" />

_Note: If your "Preview Data" fails and you get an "The access token is invalid" error, it is likely that the token expired... simply Debug to get a new token_

* Review the response and then close the "**Preview data**" window 

#### "Sink" tab

* Click on the **Sink** tab

  <img src="https://user-images.githubusercontent.com/44923999/188209702-056fff32-982b-4f9f-8d71-d0f5928db823.png" width="800" title="Snipped: September 2, 2022" />

* Complete the form by selecting your Data Explorer dataset

#### "Mapping" tab

* Click on the **Mapping** tab
* Click on "**Import schemas**"

  <img src="https://user-images.githubusercontent.com/44923999/188222315-0982b771-9a0f-4198-94c1-8e1610672d05.png" width="800" title="Snipped: September 2, 2022" />

* The resulting "**Please provide**..." pop-out should still have the Bearer Token value in the **Value** textbox from when we completed the "Preview Data" step... update as needed and then click **OK** 

  <img src="https://user-images.githubusercontent.com/44923999/188227129-5962a206-fc75-46ae-8b38-80127b6075ca.png" width="800" title="Snipped: September 2, 2022" />

* Turn on "**Advanced editor**" 
* Select "**Collection reference**" dropdown value, "**$['properties']['rows']**" 
* Click "**+ New mapping**" and enter values for each of the four columns:

  | Name | Column Name  |
  | ---- | ------------ |
  | [0]  | PreTaxCost   |
  | [1]  | UsageDate    |
  | [2]  | ResourceType |
  | [3]  | Currency     |

* Click "**Publish all**", review changes on the resulting "**Publish all**" pop-out and then click **Publish**

### Step 6: Confirm Success
In this step, we will process the pipeline and review the resulting data.

* Click **Debug** and confirm successful pipeline processing
* Navigate to your Data Explorer Database, and then **Query** in left-hand navigation
* **Run** the following KQL:
  ```
  CostManagement
  | take 10
  ```
  
  <img src="https://user-images.githubusercontent.com/44923999/188231421-c41a80d9-db55-426f-8eb3-1974a049e03e.png" width="800" title="Snipped: September 2, 2022" />

* Confirm results

  <img src="https://user-images.githubusercontent.com/44923999/187472753-de7b0a75-cea5-4ae0-af73-4117b65fa92d.png" width="200" title="Congratulations... you have successfuly completed this exercise!" />

## ...via Logic App

![image](https://user-images.githubusercontent.com/44923999/192547308-676da706-ba85-49cc-9c6e-d8bbe997fa62.png)

This use case considers requirement statements like:
* "We need to capture and analyze cost data from many subscriptions"
* "Our subscriptions have more than 1,000 resources and are hitting the Cost Management API's per-request limitation"
* "We want to pull historical data... 730 days {i.e., 2 years}"

<br>The solution described in this documentation will:
* Leverage Logic Apps' nested iteration capability with input parameters for Subscriptions and Start / End Dates 
* Request data from the Cost Management API at the Resource Group level rather than Subscription level

_Note: If you have Resource Groups with more than 1,000 resources, you might have to choose a more granular scope or other strategy_

### Step 1b: Prepare Infrastructure
This solution requires the following resources:

* [**Logic App**](Infrastructure_LogicApp.md)

### Step 2: Prepare Workflow
In this step, we will create a workflow, initialize variables, and add parameters.

* Navigate to your Logic App

  <img src="https://user-images.githubusercontent.com/44923999/190197666-84f0e96f-72c3-4ab7-b527-890eeebc0c23.png" width="800" title="Snipped: September 15, 2022" />

#### Workflow

* Click on "**Create workflow >**" in the "**Create a workflow in Designer**" rectangle
* On the resulting page click "**+ Add**"

  <img src="https://user-images.githubusercontent.com/44923999/192587724-6af8e35b-406a-4b4c-bd78-0e38c7cab16e.png" width="800" title="Snipped: September 27, 2022" />

* Complete the resulting "**New workflow**" pop-out form and then click **Create**

  _Note: Choice of stateful or stateless will depend on your use case and environment... reference: https://docs.microsoft.com/en-us/azure/logic-apps/single-tenant-overview-compare#stateful-stateless_

* Click on the link for your new workflow

  <img src="https://user-images.githubusercontent.com/44923999/192587935-a0ccf68a-4cc7-420f-ac7f-4e021ea91315.png" width="800" title="Snipped: September 27, 2022" />

* Navigate to **Designer**

  <img src="https://user-images.githubusercontent.com/44923999/192588119-475c52a0-49ce-4e8c-99c1-a098868a84df.png" width="800" title="Snipped: September 27, 2022" />

* On the resulting screen and "**Add a trigger**" pop-out, search for and then select "**Schedule**" (aka "Recurrence")

  <img src="https://user-images.githubusercontent.com/44923999/192588288-fbf2a72a-04d7-4344-acf0-9588155ba347.png" width="800" title="Snipped: September 27, 2022" />

* Complete the resulting pop-out form and then click **Save**

  _Note: I chose daily because I believe that that is how often Cost Management data updates_

#### Initialize Variable, Date

* Click the **+** icon and then "**Add an action**" on the resulting pop-up menu

  <img src="https://user-images.githubusercontent.com/44923999/192591045-fe09d81e-5ee8-41f9-b688-2d653dfd22fb.png" width="800" title="Snipped: September 27, 2022" />

* On the resulting "**Add an action**" pop-out, search for and then select "**Initialize variable**"

  <img src="https://user-images.githubusercontent.com/44923999/192591237-71d8d320-7131-4e15-b535-fff6d43c763d.png" width="800" title="Snipped: September 27, 2022" />

* Complete the resulting "**Initialize variable**" pop-out form, **Parameters** tab, including:

  | Prompt    | Entry           |
  | --------- | --------------- |
  | **Name**  | Enter "Date"    |
  | **Type**  | Select "String" |
  | **Value** | {null}          |
  
  _Note: Logic Apps does not have a data type for DateTime, so we use string and handle usage in expressions_

#### Initialize Variable, Scope

<img src="https://user-images.githubusercontent.com/44923999/192591752-1252451c-17e5-44e5-8e01-9163214d0672.png" width="800" title="Snipped: September 27, 2022" />

* Repeat for string variable **Scope**

#### Initialize Variable, KQL

<img src="https://user-images.githubusercontent.com/44923999/192592437-11340df0-697a-4a13-9c6d-5948d07e30a9.png" width="800" title="Snipped: September 27, 2022" />

* Repeat for string variable **KQL**
  
#### Parameters

* Click **Parameters** in the menu bar
* On the resulting **Parameters** pop-out form, click "**+ Create parameter**"

  <img src="https://user-images.githubusercontent.com/44923999/192592869-da6ac1f7-19fe-4cf7-901c-6c43e35fa7ed.png" width="800" title="Snipped: September 27, 2022" />

* Complete the pop-out form, including:

  | Prompt    | Entry                                                        |
  | --------- | ------------------------------------------------------------ |
  | **Name**  | Enter "Subscriptions"                                        |
  | **Type**  | Select "Array"                                               |
  | **Value** | Your SubscriptionId values in form: `[ "{Subscription1_Id}","{Subscription2_Id}" ]` |

  <img src="https://user-images.githubusercontent.com/44923999/192593500-0d272e61-1125-428b-a67a-801c0d34debd.png" width="800" title="Snipped: September 27, 2022" />

* Repeat for parameter **StartDate** 
  | Prompt    | Entry                                                        |
  | --------- | ------------------------------------------------------------ |
  | **Name**  | Enter "StartDate"                                            |
  | **Type**  | Select "String"                                              |
  | **Value** | Enter `2022-01-01` (or a date value that is meaningful for you) |

  _Note: Date values will be required in ISO-8601 formatted strings {e.g., 2022-09-15T00:00:00.0000000}; abbreviated versions {e.g., 2022-09-15} work fine_

* Repeat for parameter **EndDate**

  | Prompt    | Entry                                                        |
  | --------- | ------------------------------------------------------------ |
  | **Name**  | Enter "EndDate"                                              |
  | **Type**  | Select "String"                                              |
  | **Value** | Enter `2022-08-31` (or a date value that is meaningful for you) |
  
  _Note: Parameters will be alphabetized regardless of the order in which you create them_

* Click **X** to close the pop-out form and then click **Save**

### Step 3: Get Bearer Token
In this step, we will request an access token from the Client Credentials Token URL and initialize a Token variable.

* Navigate to **Designer**

#### HTTP, Get Token

* Click the **+** icon underneath "**Recurrence**"

  <img src="https://user-images.githubusercontent.com/44923999/192594246-6a59769f-cd1b-440c-95e0-d82620a5ec0e.png" width="800" title="Snipped: September 27, 2022" />

* Click "**Add a parallel branch**" on the resulting pop-up menu

  <img src="https://user-images.githubusercontent.com/44923999/192594362-2a0deb2e-0f92-47dd-a178-cf3edbf3b839.png" width="800" title="Snipped: September 27, 2022" />

* On the resulting "**Add an action**" pop-out, search for and then select "**HTTP**"

  <img src="https://user-images.githubusercontent.com/44923999/192596099-785e7bc4-a984-481d-9aa3-f715afef92b3.png" width="800" title="Snipped: September 27, 2022" />

* Complete the resulting "**HTTP**" pop-out form, **Parameters** tab, including:

  | Prompt      | Entry                                                        |
  | ----------- | ------------------------------------------------------------ |
  | **Method**  | Select "POST"                                                |
  | **URI**     | Enter `https://login.microsoftonline.com/16b3c013-d300-468d-ac64-7eda0820b6d3/oauth2/token` |
  | **Headers** | Add `content-type` :: `application/x-www-form-urlencoded`    |
  | **Body**    | Enter  `grant_type=client_credentials&client_id={Client Identifier}&client_secret={Client Secret}&resource=https://management.azure.com/` |

  _Note: If you expect processing to take longer than an hour {i.e., the lifespan of a token}, you might consider moving token handling to the nested iteration_

#### Initialize Variable, Token

* Click the **+** icon under **HTTP** and then "**Add an action**" on the resulting pop-up menu
* On the resulting "**Add an action**" pop-out, search for and then select "**Initialize variable**"

  <img src="https://user-images.githubusercontent.com/44923999/192597248-aae13dce-04a6-4498-831c-59050c9cd278.png" width="800" title="Snipped: September 27, 2022" />

* Complete the resulting "**Initialize variable**" pop-out form, **Parameters** tab, including:

  | Prompt    | Entry                                                        |
  | --------- | ------------------------------------------------------------ |
  | **Name**  | Enter "Token"                                                |
  | **Type**  | Select "String"                                              |
  | **Value** | Enter expression:<br>`concat('Bearer ',body('HTTP,_Get_Token').access_token)`<br><br>...and then click **OK** |

  _Note: The parenthetic "body" value {i.e., `body('HTTP,_Get_Token')`} may need to change depending on what you named the component_ 

* Click **Save**

#### Confirm Success

* Navigate to **Overview**, click "**Run Trigger**", and then click **Run** in the resulting dropdown menu

  <img src="https://user-images.githubusercontent.com/44923999/192598688-069b7473-6a9c-4b5e-bc37-c77f3814dc4f.png" width="800" title="Snipped: September 27, 2022" />

* Confirm that all actions succeed and click on those you would like to understand better

### Step 4: Prepare Dates Array
In this step, we will iterate through dates between StartDate and EndDate and append to the Dates array.

* Navigate to **Designer**

#### Initialize Variable, Counter

* Click the **+** icon underneath "**Recurrence**" and then "**Add a parallel branch**" on the resulting pop-up menu
* On the resulting "**Add an action**" pop-out, search for and then select "**Initialize variable**"

  <img src="https://user-images.githubusercontent.com/44923999/192600226-de668e6a-fa41-44ea-85a3-032b67d9e88c.png" width="800" title="Snipped: September 27, 2022" />

* Complete the resulting "**Initialize variable**" pop-out form, **Parameters** tab, including:

  | Prompt    | Entry            |
  | --------- | ---------------- |
  | **Name**  | Enter "Counter"  |
  | **Type**  | Select "Integer" |
  | **Value** | Enter 0          |

#### Initialize Variable, Dates

  <img src="https://user-images.githubusercontent.com/44923999/192601399-1cf57b86-47cb-4678-a2d7-d8dcf36c7592.png" width="800" title="Snipped: September 27, 2022" />

* Repeat for array variable **Dates** (with no initial value)

#### Do..Until

* Click the **+** icon and then "**Add an action**" on the resulting pop-up menu

  <img src="https://user-images.githubusercontent.com/44923999/192621146-85ff665a-e906-4b07-b78e-68a04206fefb.png" width="800" title="Snipped: September 27, 2022" />

* On the resulting "**Add an action**" pop-out, search for and then select "**Until**"

  <img src="https://user-images.githubusercontent.com/44923999/192621503-61143154-dd17-43d2-aefe-62c0c18cc0d8.png" width="800" title="Snipped: September 27, 2022" />

* Complete the resulting **Until** pop-out form, **Parameters** tab, including:

  | Prompt             | Entry                                                        |
  | ------------------ | ------------------------------------------------------------ |
  | **Choose a value** | Enter expression:<br>`addDays(parameters('StartDate'), variables('Counter'))` |
  | **Type**           | Select "is greater than"                                     |
  | **Choose a value** | Enter expression:<br>`addDays(parameters('EndDate'), 0)`     |

#### Append to Array Variable, Date

* Click the **+** icon inside the "**Do..Until**" action and then "**Add an action**" on the resulting pop-up menu

  <img src="https://user-images.githubusercontent.com/44923999/192622104-b9b829e1-115a-465b-8188-5aa8eb7ce3ca.png" width="800" title="Snipped: September 27, 2022" />

* On the resulting "**Add an action**" pop-out, search for and then select "**Append to array variable**"

  <img src="https://user-images.githubusercontent.com/44923999/192623210-b34b6282-152f-4df1-a5d6-4637a1e3ff2c.png" width="800" title="Snipped: September 27, 2022" />

* Complete the resulting "**Append to array variable**" pop-out form, **Parameters** tab, including:

  | Prompt    | Entry                                                        |
  | --------- | ------------------------------------------------------------ |
  | **Name**  | Select "Dates"                                               |
  | **Value** | Enter expression:<br>`addDays(parameters('StartDate'),variables('Counter'))` |
  
#### Increment Variable, Counter

* Click the **+** icon inside the "**Do..Until**" action and then "**Add an action**" on the resulting pop-up menu

  <img src="https://user-images.githubusercontent.com/44923999/192623419-4816edf7-8eba-40ed-95ec-82f956688289.png" width="800" title="Snipped: September 27, 2022" />

* On the resulting "**Add an action**" pop-out, search for and then select "**Increment variable**"

  <img src="https://user-images.githubusercontent.com/44923999/192623566-0b7f5a84-155c-4f6d-8e9f-977d37805a53.png" width="800" title="Snipped: September 27, 2022" />

* Complete the resulting "**Append to array variable**" pop-out form, **Parameters** tab, including:

  | Prompt    | Entry            |
  | --------- | ---------------- |
  | **Name**  | Select "Counter" |
  | **Value** | Enter "1"        |
  
* Click **Save**

#### Confirm Success

* Navigate to **Overview**, click "**Run Trigger**", and then click **Run** in the resulting dropdown menu
* Click on the new "**Running**" item in the "**Run History**" list

  <img src="https://user-images.githubusercontent.com/44923999/192623862-67e1bccb-a259-43f9-b97c-304dba148971.png" width="800" title="Snipped: September 27, 2022" />

* Confirm that all actions succeed and click on those you would like to understand better

### Step 5: Iterate through Subscriptions
In this step, we will create a "For Each" action for Subscriptions.

* Navigate to **Designer**

#### For Each, Subscription

* Click the **+** icon at the bottom of the page and then "**Add an action**" on the resulting pop-up menu

  <img src="https://user-images.githubusercontent.com/44923999/192624266-b5b3a4d7-2d5a-4a7f-9064-70634dbf7290.png" width="800" title="Snipped: September 27, 2022" />

  _Note: Dependencies from all parallel branches will be added to the new action_

* On the resulting "**Add an action**" pop-out, search for and then select "**For each**"

  <img src="https://user-images.githubusercontent.com/44923999/192624716-e3e20dc5-ec5a-4614-a160-f3e6c310018c.png" width="800" title="Snipped: September 27, 2022" />

* Complete the resulting "**For each**" pop-out form, **Parameters** tab, including:

  | Prompt                                   | Entry                  |
  | ---------------------------------------- | ---------------------- |
  | **Select an output from previous steps** | Select "Subscriptions" |

#### HTTP, Get Resource Groups

* Click the **+** icon inside the "**For Each, Subscription**" action and then "**Add an action**" on the resulting pop-up menu
* On the resulting "**Add an action**" pop-out, search for and then select "**HTTP**"

  <img src="https://user-images.githubusercontent.com/44923999/192627282-3ac16593-cdfb-4319-b721-4434907849fa.png" width="800" title="Snipped: September 27, 2022" />

* Complete the resulting "**HTTP**" pop-out form, **Parameters** tab, including:

  | Prompt      | Entry                                                        |
  | ----------- | ------------------------------------------------------------ |
  | **Method**  | Select "GET"                                                 |
  | **URI**     | Enter expression:<br>`concat('https://management.azure.com/subscriptions/',item(),'/resourcegroups?api-version=2021-04-01')` |
  | **Headers** | Add headers:<br>`Authorization` :: `variables('Token')`<br>`content-type` :: `application/json;charset=utf-8` |

  _Note: Capitalization of the header key "Authorization" appears to be important to this API_

* Click **Save**

#### Confirm Success

* Navigate to **Overview**, click "**Run Trigger**", and then click **Run** in the resulting dropdown menu
* Click on the new "**Running**" item in the "**Run History**" list

  <img src="https://user-images.githubusercontent.com/44923999/192626982-a02a11f7-9ffb-4d82-af51-24b502cbb331.png" width="800" title="Snipped: September 27, 2022" />

* Confirm that all actions succeed and click on those you would like to understand better

### Step 6: Iterate through Resource Groups
In this step, we will create a "For Each" action for Resource Groups {aka Scopes}.

* Navigate to **Designer**

#### For Each, Resource Group

* Click the **+** icon inside the "**For Each, Subscription**" action and then "**Add an action**" on the resulting pop-up menu
* On the resulting "**Add an action**" pop-out, search for and then select "**For each**"

  <img src="https://user-images.githubusercontent.com/44923999/192627843-ff8a80b0-0422-4ff1-8017-a3925cfb0608.png" width="800" title="Snipped: September 27, 2022" />

* Complete the resulting "**For each**" pop-out form, **Parameters** tab, including:

  | Prompt                                   | Entry                                                       |
  | ---------------------------------------- | ----------------------------------------------------------- |
  | **Select an output from previous steps** | Enter expression: `body('HTTP,_Get_Resource_Groups').value` |

#### Set Variable, Scope

* Click the **+** icon inside the "**For Each, Resource Group**" action and then "**Add an action**" on the resulting pop-up menu
* On the resulting "**Add an action**" pop-out, search for and then select "**Set variable**"

  <img src="https://user-images.githubusercontent.com/44923999/192628042-c247947c-cd7c-45b1-ab6b-c52e4c2a22c1.png" width="800" title="Snipped: September 27, 2022" />

* Complete the resulting "**Set variable**" pop-out form, **Parameters** tab, including:

  | Prompt    | Entry                         |
  | --------- | ----------------------------- |
  | **Name**  | Select "**Scope**"            |
  | **Value** | Enter expression: `item().id` |

* Click **Save**

#### Confirm Success

* Navigate to **Overview**, click "**Run Trigger**", and then click **Run** in the resulting dropdown menu
* Click on the new "**Running**" item in the "**Run History**" list

  <img src="https://user-images.githubusercontent.com/44923999/192628455-6d963fd8-3e08-42a1-b925-b739fbdbd4f7.png" width="800" title="Snipped: September 27, 2022" />

* Confirm that all actions succeed and click on those you would like to understand better

### Step 7: Iterate through Dates
In this step, we will nest "For Each" actions for Dates.

* Navigate to **Designer**

#### For Each, Date

* Click the **+** icon inside the "**For Each, Resource Group**" action and then "**Add an action**" on the resulting pop-up menu
* On the resulting "**Add an action**" pop-out, search for and then select "**For each**"

  <img src="https://user-images.githubusercontent.com/44923999/192628876-d9b264c4-d347-496b-ac71-2b2f9c8f1798.png" width="800" title="Snipped: September 27, 2022" />

* Complete the resulting "**For each**" pop-out form, **Parameters** tab, including:

  | Prompt                                   | Entry                                  |
  | ---------------------------------------- | -------------------------------------- |
  | **Select an output from previous steps** | Enter expression: `variables('Dates')` |

#### Set Variable, Date

* Click the **+** icon inside the "**For Each, Date**" action and then "**Add an action**" on the resulting pop-up menu
* On the resulting "**Add an action**" pop-out, search for and then select "**Set variable**"

  <img src="https://user-images.githubusercontent.com/44923999/192629234-e12c2485-e04c-4102-9dea-fdba5d50ae8c.png" width="800" title="Snipped: September 27, 2022" />

* Complete the resulting "**Set variable**" pop-out form, **Parameters** tab, including:

  | Prompt    | Entry                      |
  | --------- | -------------------------- |
  | **Name**  | Select "Date"              |
  | **Value** | Enter expression: `item()` |

* Click **Save**

#### Confirm Success

* Navigate to **Overview**, click "**Run Trigger**", and then click **Run** in the resulting dropdown menu
* Click on the new "**Running**" item in the "**Run History**" list

  <img src="https://user-images.githubusercontent.com/44923999/192630220-9f3fbbd4-b60a-4871-923d-e05cbb8f42de.png" width="800" title="Snipped: September 27, 2022" />

* Confirm that all actions succeed and click on those you would like to understand better

### Step 8: Get Cost Data
In this step, we will request and process data from the Cost Management API.

* Navigate to **Designer**

#### HTTP, Get Costs

* Click the **+** icon inside the "**For Each, Date**" action and then "**Add an action**" on the resulting pop-up menu
* On the resulting "**Add an action**" pop-out, search for and then select "**HTTP**"

  <img src="https://user-images.githubusercontent.com/44923999/192630740-f07210f9-c467-4779-bddd-62bb69af05fe.png" width="800" title="Snipped: September 27, 2022" />

* Complete the resulting "**HTTP**" pop-out form, **Parameters** tab, including:

  | Prompt      | Entry                                                        |
  | ----------- | ------------------------------------------------------------ |
  | **Method**  | Select "POST"                                                |
  | **URI**     | Enter expression:<br>`https://management.azure.com/@{variables('Scope')}/providers/Microsoft.CostManagement/query?api-version=2021-10-01` |
  | **Headers** | Add headers:<br>`authorization` :: `variables('Token')`<br>`content-type` :: `application/json;charset=utf-8` |

* Finally, paste the following in **Body**:

  ```
  {
    "dataset": {
      "aggregation": {
        "totalCost": {
          "function": "Sum",
          "name": "PreTaxCost"
        }
      },
      "granularity": "Daily",
      "grouping": [
        {
          "name": "ResourceGroupName",
          "type": "Dimension"
        },
        {
          "name": "ResourceType",
          "type": "Dimension"
        },
        {
          "name": "ResourceId",
          "type": "Dimension"
        },
        {
          "name": "ResourceLocation",
          "type": "Dimension"
        },
        {
          "name": "MeterCategory",
          "type": "Dimension"
        },
        {
          "name": "MeterSubCategory",
          "type": "Dimension"
        },
        {
          "name": "Meter",
          "type": "Dimension"
        },
        {
          "name": "ServiceName",
          "type": "Dimension"
        },
        {
          "name": "PartNumber",
          "type": "Dimension"
        },
        {
          "name": "PricingModel",
          "type": "Dimension"
        },
        {
          "name": "ChargeType",
          "type": "Dimension"
        },
        {
          "name": "ReservationName",
          "type": "Dimension"
        },
        {
          "name": "Frequency",
          "type": "Dimension"
        }
      ]
    },
    "timePeriod": {
      "from": "@{variables('Date')}",
      "to": "@{variables('Date')}"
    },
    "timeframe": "Custom",
    "type": "Usage"
  }
  ```

  _Note: Scope ResourceGroup does not allow use of **BillingPeriod** and **ServiceTier** columns_

#### Parse JSON

* Click the **+** icon inside the "**For Each, Date**" action and then "**Add an action**" on the resulting pop-up menu
* On the resulting "**Add an action**" pop-out, search for and then select "**Parse JSON**"

  <img src="https://user-images.githubusercontent.com/44923999/192637507-42df62f5-22e8-4ae0-988d-2e7db949b631.png" width="800" title="Snipped: September 27, 2022" />

* On the resulting "**Parse JSON**" pop-out form, **Parameters** tab, **Content** textbox, select dynamic content "Body" from the "**HTTP, Get Costs**" grouping and then, paste the following in **Schema**:

  ```
  {
      "properties": {
          "eTag": {},
          "id": {
              "type": "string"
          },
          "location": {},
          "name": {
              "type": "string"
          },
          "properties": {
              "properties": {
                  "columns": {
                      "items": {
                          "properties": {
                              "name": {
                                  "type": "string"
                              },
                              "type": {
                                  "type": "string"
                              }
                          },
                          "required": [
                              "name",
                              "type"
                          ],
                          "type": "object"
                      },
                      "type": "array"
                  },
                  "nextLink": {},
                  "rows": {
                      "items": {
                          "type": "array"
                      },
                      "type": "array"
                  }
              },
              "type": "object"
          },
          "sku": {},
          "type": {
              "type": "string"
          }
      },
      "type": "object"
  }
  ```

* Click **Save**

#### Confirm Success

* Navigate to **Overview**, click "**Run Trigger**", and then click **Run** in the resulting dropdown menu
* Click on the new "**Running**" item in the "**Run History**" list

  <img src="https://user-images.githubusercontent.com/44923999/192638440-104b408d-96d8-4271-b304-a94c4c5b777c.png" width="800" title="Snipped: September 27, 2022" />

* Confirm that all actions succeed and click on those you would like to understand better

### Step 9: Ingest to Data Explorer
In this step, we will send the Cost Management API response to Data Explorer using an `.ingest inline` command.

* Navigate to **Designer**

#### For Each, Response Row

* Click the **+** icon inside the "**For Each, Date**" action and below the "**Parse JSON**" action
* Then click "**Add an action**" on the resulting pop-up menu
* On the resulting "**Add an action**" pop-out, search for and then select "**For each**"

  <img src="https://user-images.githubusercontent.com/44923999/192640350-baaa1637-a1c9-43c8-9ae3-89cd66917b53.png" width="800" title="Snipped: September 27, 2022" />

* On the resulting "**For each**" pop-out form, **Parameters** tab, "**Select an output from previous steps**" textbox, select dynamic content "**rows**" from the "**Parse JSON**" grouping

#### Set Variable, KQL

* Click the **+** icon inside the "**For Each, Resource Group**" action and then "**Add an action**" on the resulting pop-up menu
* On the resulting "**Add an action**" pop-out, search for and then select "**Set variable**"

  <img src="https://user-images.githubusercontent.com/44923999/192789432-8591ac03-e7b3-496d-a7cd-588cb5ad3be2.png" width="800" title="Snipped: September 28, 2022" />

* Complete the resulting "**Set variable**" pop-out form, **Parameters** tab, including:

  | Prompt    | Entry                                                        |
  | --------- | ------------------------------------------------------------ |
  | **Name**  | Select "**KQL**"                                             |
  | **Value** | Enter expression: `concat('.ingest inline into table CostManagement <| ', replace(replace(string(item()),'[',''),']',''))` |

* Click **Save**

#### ADX Command, Ingest

* Click the **+** icon inside the "**For Each, Response Row**" action and then "**Add an action**" on the resulting pop-up menu
* On the resulting "**Add an action**" pop-out, click the **Azure** tab, search for and then select "**Run control command and render a chart**"

  <img src="https://user-images.githubusercontent.com/44923999/192791316-11fd9dec-a0bb-467d-8c08-4acd340e4ca8.png" width="800" title="Snipped: September 28, 2022" />

* Complete the resulting "**Run control command and render a chart**" pop-out form, **Parameters** tab, including:

  | Prompt              | Entry                   |
  | ------------------- | ----------------------- |
  | **Control Command** | Select variable **KQL** |
  | **Chart Type**      | Select "**Html Table**" |
  
  _Note: the selected "**Chart Type**" value does not matter; it is required by the Operation, but the result will not be used_

* Click **Save**

#### Confirm Success

* Navigate to **Overview**, click "**Run Trigger**", and then click **Run** in the resulting dropdown menu
* Click on the new "**Running**" item in the "**Run History**" list

  <img src="https://user-images.githubusercontent.com/44923999/192792427-f3052503-5ea6-4924-9d79-aa62bd827847.png" width="800" title="Snipped: September 28, 2022" />

* Confirm that all actions succeed and click on those you would like to understand better

### Step 10: Query Data

* Navigate to Data Explorer Database, then Query in the Data grouping of the left-hand navigation

  <img src="https://user-images.githubusercontent.com/44923999/192793220-ea2cc1b9-d35a-4255-b591-a2c2892d55ae.png" width="800" title="Snipped: September 28, 2022" />

* Run the following KQL:

  ```
  CostManagement
  | extend IngestionTime = ingestion_time()
  | sort by IngestionTime desc
  ```

  <img src="https://user-images.githubusercontent.com/44923999/187472753-de7b0a75-cea5-4ae0-af73-4117b65fa92d.png" width="200" title="Congratulations... you have successfuly completed this exercise!" />

## ...via Function App

```
using Microsoft.Azure.Management.Fluent;
using Microsoft.Azure.Management.ResourceManager.Fluent;
using System.Net.Http.Headers;
using System.Text;
using System.Text.Json;

public class Program
{
    public class OAuth2
    {
        public string? access_token { get; set; }
    }

    public static async Task Main(string[] args)
    {
        Console.WriteLine("Start: " + DateTime.Now);

        string clientid = "{YOUR CLIENT ID}",
            clientsecret = "{YOUR CLIENT SECRET}",
            tenantid = "{YOUR TENANT ID}",
            startDate = "11/4/2022",
            endDate = "11/5/2022";

        string[] subscriptions = new string[] { "{YOUR SUBSCRIPTION ID}", "{YOUR SUBSCRIPTION ID}" };

        var credentials = SdkContext.AzureCredentialsFactory.FromServicePrincipal(
            clientId: clientid,
            clientSecret: clientsecret,
            tenantId: tenantid,
            environment: AzureEnvironment.AzureGlobalCloud
        );

        /* ************************* Iterate Subscriptions */

        foreach (string subscription in new string[] { "ed7eaf77-d411-484b-92e6-5cba0b6d8098" })
        {
            var azure = Azure
                .Authenticate(credentials)
                .WithSubscription(subscriptionId: subscription)
                ;

            /* ************************* Iterate Resource Groups */

            foreach (var resourceGroup in azure.ResourceGroups.List())
            {
                /* ************************* Get Token */

                string token;

                using (HttpClient client = new HttpClient())
                {
                    FormUrlEncodedContent content = new FormUrlEncodedContent(new[]{
                        new KeyValuePair<string, string>("grant_type", "client_credentials"),
                        new KeyValuePair<string, string>("client_id", clientid),
                        new KeyValuePair<string, string>("client_secret", clientsecret),
                        new KeyValuePair<string, string>("resource", "https://management.azure.com/")
                    });

                    content.Headers.ContentType = new MediaTypeHeaderValue("application/x-www-form-urlencoded");

                    HttpResponseMessage response = await client.PostAsync(requestUri: new Uri("https://login.microsoftonline.com/16b3c013-d300-468d-ac64-7eda0820b6d3/oauth2/token"), content: content);

                    OAuth2? oauth2 = JsonSerializer.Deserialize<OAuth2>(response.Content.ReadAsStringAsync().Result);

                    token = oauth2.access_token;
                }

                /* ************************* Iterate Dates */

                for (var date = DateTime.Parse(startDate); date <= DateTime.Parse(endDate); date = date.AddDays(1))
                {
                    Console.WriteLine(Environment.NewLine + resourceGroup.Id + " >> " + date.ToString("MMMM dd yyyy"));

                    /* ************************* Request Cost Management Data */

                    using (HttpClient client = new HttpClient())
                    {
                        client.DefaultRequestHeaders.Add("Authorization", "Bearer " + token);

                        HttpRequestMessage request = new HttpRequestMessage(
                            method: new HttpMethod("POST"),
                            requestUri: "https://management.azure.com/" + resourceGroup.Id + "/providers/Microsoft.CostManagement/query?api-version=2021-10-01");

                        request.Content = new StringContent(
                            content: "{'dataset':{'aggregation':{'totalCost':{'function':'Sum','name':'PreTaxCost'}},'granularity':'Daily','grouping':[{'name':'ResourceGroupName','type':'Dimension'},{'name':'ResourceType','type':'Dimension'},{'name':'ResourceId','type':'Dimension'},{'name':'ResourceLocation','type':'Dimension'},{'name':'MeterCategory','type':'Dimension'},{'name':'MeterSubCategory','type':'Dimension'},{'name':'Meter','type':'Dimension'},{'name':'ServiceName','type':'Dimension'},{'name':'PartNumber','type':'Dimension'},{'name':'PricingModel','type':'Dimension'},{'name':'ChargeType','type':'Dimension'},{'name':'ReservationName','type':'Dimension'},{'name':'Frequency','type':'Dimension'}]},'timePeriod':{'from':'" + date + "','to':'" + date + "'},'timeframe':'Custom','type':'Usage'}",
                            encoding: Encoding.UTF8,
                            mediaType: "application/json");

                        HttpResponseMessage response = client.Send(request);

                        Console.WriteLine(response.Content.ReadAsStringAsync().GetAwaiter().GetResult());
                    }
                }
            }
        }
        Console.WriteLine("End: " + DateTime.Now);
    }
}
```