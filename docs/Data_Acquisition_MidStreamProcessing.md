# Data Acquisition: Mid-Stream Processing

![image](https://user-images.githubusercontent.com/44923999/208464152-33914e21-5ae5-49fc-8a9a-dc7dddcf0339.png)

## Use Case

This solution considers the following requirements:

* "We stream millions of messages per hour from an Event Hub owned and controlled by another organization"
* "Each message requires special, mid-stream handling {e.g., format translation, decompression, re-packing of JSON, etc.}"

## Proposed Solution

This solution will address requirements in three steps:

* Mimic Source - For this exercise, we must loosely mirror the source Event Hub... in the real-world, we would just connect to the real source
* Process Mid-Stream - create a Function App with incoming Event Hub (source), processing logic {e.g. unpack JSON} and outgoing Event Hub (destination)
* Ingest Data - ingest data into Data Explorer

## Prepare Infrastructure

This solution requires the following resources:

* Data Explorer [**Cluster**](Infrastructure_DataExplorer_Cluster.md) and [**Database**](Infrastructure_DataExplorer_Database.md)
* [**Event Hub**](https://learn.microsoft.com/en-us/azure/event-hubs/)... one namespace with two event hubs (incoming and outgoing)
* [**Function App**](https://learn.microsoft.com/en-us/azure/azure-functions/functions-overview) configured to use [**Application Insights**](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview) for monitoring
* [Visual Studio](https://visualstudio.microsoft.com/) with **Azure development** workload

## Exercise 1: Mimic Source

In this exercise, we will use a Function App to mock the flow of messages coming from the source Event Hub.

### Step 1: Create Visual Studio Project

* Open Visual Studio

  <img src="https://user-images.githubusercontent.com/44923999/201695714-2c2be122-0b23-42ef-8a47-f71a4bc3ac76.png" width="800" title="Snipped: November 14, 2022" />

* Click "**Create a new project**"

  <img src="https://user-images.githubusercontent.com/44923999/201696928-02adc19a-6cfe-45b9-8271-eb71da588f0d.png" width="800" title="Snipped: November 14, 2022" />

* On the "**Create a new project**" page, search for and select "**Azure Functions**", then click **Next**

  <img src="https://user-images.githubusercontent.com/44923999/208470843-1698307e-c955-4c60-9173-798460baa275.png" width="800" title="Snipped: December 19, 2022" />

* Complete the "**Configure your new project**" form and then click **Next**

  <img src="https://user-images.githubusercontent.com/44923999/201709567-ea5c59be-014e-4f38-9781-582afd44edea.png" width="800" title="Snipped: November 14, 2022" />

* Complete the "**Additional information**" form, including:

  | Prompt               | Entry                                                        |
  | -------------------- | ------------------------------------------------------------ |
  | **Functions worker** | Select "**.NET 6.0 (Long Term Support)**"                    |
  | **Function**         | Select "**Timer trigger**"                                   |
  | **Use Azurite...**   | Checked                                                      |
  | **Schedule**         | Enter "***/1 * * * ***" (the CRON expression for every one minute) |

* Click **Create**

### Step 2: "Function1.cs" Logic, Update Method

* Open "Function1.cs"

  <img src="https://user-images.githubusercontent.com/44923999/208492132-99ce75ef-18f5-416a-8852-c34d37d35a09.png" width="800" title="Snipped: December 19, 2022" />

* Replace the default code `public void Run([TimerTrigger("*/1 * * * *")]TimerInfo myTimer, TraceWriter log)` with:

  ```
  public async Task Run(
    [TimerTrigger("*/1 * * * *")] TimerInfo theTimer,
    [EventHub("dest", Connection = "EventHubConnectionAppSetting")] IAsyncCollector<string> theEventHub,
    ILogger theLogger)
  ```

  Logic Explained:

  * `async Task` ... provides for use of asynchronous calls
  * `[EventHub...` ... provides for **output** to Event Hub

### Step 3: Install NuGet

* In the "**Error List**", you will notice errors like "The type or namespace 'EventHub' could not be found..."; these must be resolved by adding a NuGet Package

  <img src="https://user-images.githubusercontent.com/44923999/208492582-5cc800c9-12f4-4ec6-9c6b-0431d56cfb4e.png" width="800" title="Snipped: December 19, 2022" />

* Click **Tools** in the menu bar, expand "**NuGet Package Manager**" in the resulting menu and then click "**Manage NuGet Packages for Solution...**"

  <img src="https://user-images.githubusercontent.com/44923999/208492799-a3c55385-e5f7-48be-843f-ee4a5efc65e9.png" width="800" title="Snipped: December 19, 2022" />

* On the **Browse** tab of the "**NuGet - Solution**" page, search for and select "**Microsoft.Azure.WebJobs.Extensions.EventHubs**"

* On the resulting pop-out, check project **MidStreamProcessing** and then click **Install**

* When prompted, click "**I Accept**" on the "**License Acceptance**" pop-up

### Step 4: Update "Function1.cs"

* Return to "Function1.cs"

  <img src="https://user-images.githubusercontent.com/44923999/208499718-fb7658ab-5dff-4d8d-9384-752f62a120f7.png" width="800" title="Snipped: December 19, 2022" />

* Replace the default code `log.LogInformation($"C# Timer trigger function executed at: {DateTime.Now}");` with:

  ```
  string output = "{\"rows\":[{\"id\":"+Guid.NewGuid()+ "},{\"id\":"+Guid.NewGuid()+"},{\"id\":"+Guid.NewGuid()+"}]}";
  
  theLogger.LogInformation(output);
  
  await theEventHub.AddAsync(output);
  ```

  Logic Explained:

  * `string output = ...` generates a nested JSON string like `{"rows":[{"id":5740c202-3ee8-43c0-8561-01a33506cb6f},{"id":78ec5ee8-95f1-4189-85db-d14a53916566},{"id":8fc61f2f-db87-4424-8860-d553983f908f}]}`
  * `await theEventHub.AddAsync(output);` sends the JSON string to the Event Hub

### Step 5: Update "local.settings.json"

* Open "local.settings.json"

  <img src="https://user-images.githubusercontent.com/44923999/208500134-9dcf4c03-4226-4ccb-b5af-32af5588b886.png" width="800" title="Snipped: December 19, 2022" />

* Modify the following JSON {e.g., replace placeholders like STORAGE_ACCOUNT_NAME with real values} and then replace default "local.settings.json" content

  ```
  {
    "IsEncrypted": false,
    "Values": {
      "AzureWebJobsStorage": "DefaultEndpointsProtocol=https;AccountName=STORAGE_ACCOUNT_NAME;AccountKey=STORAGE_ACCOUNT_KEY;EndpointSuffix=core.windows.net",
      "EventHubConnectionAppSetting": "Endpoint=sb://EVENTHUB_NAMESPACE_NAME.servicebus.windows.net/;SharedAccessKeyName=EVENTHUB_SHAREDACCESSPOLICY_NAME;SharedAccessKey=EVENTHUB_SHAREDACCESSPOLICY_KEY;EntityPath=EVENTHUB_NAME",
      "FUNCTIONS_WORKER_RUNTIME": "dotnet"
    }
  }
  ```

### Step 6: Confirm Success (local)

* In the Visual Studio menu-bar, click **Debug** >> "**Start Debugging**"

  <img src="https://user-images.githubusercontent.com/44923999/208506217-b908f61d-d67d-4e54-8c94-14de07bf6685.png" width="800" title="Snipped: December 19, 2022" />

* In the resulting Command window, monitor processing and confirm success

  <img src="https://user-images.githubusercontent.com/44923999/208505976-69e389b4-2850-44b5-8e17-8728d4bfc6dd.png" width="800" title="Snipped: December 19, 2022" />

* Navigate to the Event Hub and confirm **Incoming Messages**

### Step 7: Publish to Azure

* Return to Visual Studio

<img src="https://user-images.githubusercontent.com/44923999/208507075-b1675f1b-0645-4460-94bb-2a8f2c14dec9.png" width="800" title="Snipped: December 19, 2022" />

* Right-click on the MidStreamProcessing project and select Publish from the resulting drop-down menu

  <img src="https://user-images.githubusercontent.com/44923999/208524701-66855dbf-aa15-4166-823b-9a3ab7335312.png" width="600" title="Snipped: December 19, 2022" />

* On the **Publish** pop-up, **Target** tab, select "**Azure**" and then click **Next**

  <img src="https://user-images.githubusercontent.com/44923999/208525720-9123df26-a749-469e-89bc-47a22ad8e997.png" width="600" title="Snipped: December 19, 2022" />

* On the **Publish** pop-up, "**Specific target**" tab, select "**Azure Function App (Windows)**" and then click **Next**

  <img src="https://user-images.githubusercontent.com/44923999/208526223-fab7fdcc-64a6-4e23-80d1-c3c20ae5c846.png" width="600" title="Snipped: December 19, 2022" />

* On the **Publish** pop-up, "**Functions instance**" tab, select your subscription, expand to and select your Function App, then click **Finish**

  <img src="https://user-images.githubusercontent.com/44923999/208526895-da1b4106-0231-4c2b-8147-4cbc2468a64f.png" width="600" title="Snipped: December 19, 2022" />

* On the **Publish** pop-up, "**Finish**" tab, monitor publish profile creation progress and then click **Close**

### Step 8: Configure Application Settings

* Navigate to the Function App, then **Configuration** in the **Settings** group of the left-hand navigation pane

  <img src="https://user-images.githubusercontent.com/44923999/208687614-b4dbd19d-1691-4227-96b0-1f51de52d3d0.png" width="800" title="Snipped: December 20, 2022" />

* Click "**+ New application setting**"

  <img src="https://user-images.githubusercontent.com/44923999/208687847-124e1f5b-6aee-4709-b135-d313731fda82.png" width="800" title="Snipped: December 20, 2022" />

* Complete the resulting "**Add/Edit application setting**" pop-out, including:

  | Prompt    | Entry                                                        |
  | --------- | ------------------------------------------------------------ |
  | **Name**  | Enter "**EventHubConnectionAppSetting**"                     |
  | **Value** | Modify and enter the following string:<br> `Endpoint=sb://EVENTHUB_NAMESPACE_NAME.servicebus.windows.net/;SharedAccessKeyName=EVENTHUB_SHAREDACCESSPOLICY_NAME;SharedAccessKey=EVENTHUB_SHAREDACCESSPOLICY_KEY;EntityPath=EVENTHUB_NAME` |

* Click **OK** to close the pop-out and then **Save** on the **Configuration** page, "**Application setting**" tab

### Step 9: Confirm Success (published to Azure)

* Navigate to Function1, then **Monitor** in the **Developer** group of the left-hand navigation pane

  <img src="https://user-images.githubusercontent.com/44923999/208690334-19be532d-75d2-4d23-b98f-eaf144bbb56f.png" width="800" title="Snipped: December 20, 2022" />

* Confirm **Success** messages

  <img src="https://user-images.githubusercontent.com/44923999/208690545-52a314ef-6999-4fde-964c-eba38768d100.png" width="800" title="Snipped: December 20, 2022" />

* Navigate to the Event Hub and confirm Incoming Messages

## Exercise 2: Process Mid-Stream

  <img src="https://user-images.githubusercontent.com/44923999/208710138-a6f25c3e-88a9-4195-aebf-54c8016d5b5c.png" width="800" title="Snipped: December 20, 2022" />

Were we to ingest the source data directly {i.e., without mid-stream processing}, our best strategy would be to pull it in Data Format "TXT", and then convert to dynamic / parse with KQL in Data Explorer.

In this exercise, we will create a Function App with incoming Event Hub (source), processing logic {e.g. unpack JSON} and outgoing Event Hub (destination)

### Step 1: Lorem Ipsum

* Open Visual Studio