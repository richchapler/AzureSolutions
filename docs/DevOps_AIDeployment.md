# DevOps: AI Deployment (UPDATE IN PROGRESS)

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/20ef5226-59b5-4876-b8b2-789373480cb4" width="1000" />

## Use Case
* "We have implemented OpenAI with AI Search and are rapidly iterating through enhancements to the index"
* "Creating and updating the AI Search index can be difficult... we want a simpler, faster, more consistent experience"
* "We want to capture our AI Search index creation process in our DevOps repo"
* "Codified AI Search skills must include: OCR, Key Phrase Extraction, and Custom Skillset"
* "All secrets must be stored in Key Vault"

## Proposed Solution
* Custom Skillset API: Use a Function App to instantiate a simple API for use with AI Search, custom skillset
* Deployment Application: Use the AI Search Development Kit (SDK) to create a data source, index, skillset, and indexer
* Source Control: Create a pull request in a DevOps repo

## Solution Requirements
* [**AI Search**](https://azure.microsoft.com/en-us/products/search) with dependency:
  * [**AI Services**](https://learn.microsoft.com/en-us/azure/cognitive-services/)
* [**DevOps**](https://azure.microsoft.com/en-us/products/devops/) with organization and project
* [**Function App**](https://learn.microsoft.com/en-us/azure/azure-functions/functions-overview) configured for .NET 7, with dependencies:
  * [Application Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview)
  * [Application Service](https://learn.microsoft.com/en-us/azure/app-service/)
  * [Storage Account](Infrastructure_StorageAccount.md)
* [**Key Vault**](https://learn.microsoft.com/en-us/azure/key-vault) with the following [secrets](https://learn.microsoft.com/en-us/azure/key-vault/secrets):
  * ConnectionString_BlobStorage
  * Key_AISearch
  * Key_AIServices
* [**Postman**](https://www.postman.com/product/workspaces/) (with Desktop Agent for localhost testing)
* [**Storage Account**](Infrastructure_StorageAccount.md) with a container and uploaded sample data {e.g., [IRS Tax Forms](https://www.irs.gov/forms-instructions)}
* [**Visual Studio**](https://visualstudio.microsoft.com/downloads/) with **Azure development** workload and connected to your DevOps project

-----

## Exercise 1: Custom Skillset API
In this exercise, we will use a Function App to instantiate a simple API for use with AI Search, custom skillset.
<br>_Note: Only complete this exercise if you intend to include a custom skillset in the AI Search deployment app_

### Step 1: Create Project

Open Visual Studio and click "**Create a new project**".

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/2b9ad75a-370d-4642-98a0-78600a43b772" width="600" title="Snipped: October 27, 2023" />

On the "**Create a new project**" page, search for and select "**Azure Functions**", then click "**Next**".

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/877158e8-9b6e-4c88-9346-307bc73095f6" width="600" title="Snipped: October 27, 2023" />

Complete the "**Configure your new project**" form and then click "**Next**".

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/39eeeab1-76b4-4325-8772-0c7aafa3c6d9" width="600" title="Snipped: October 27, 2023" />

Complete the "**Additional information**" form:

Prompt | Entry
:----- | :-----
**Functions worker** | **.NET 7.0 Isolated**
**Function** | **Http trigger**
**Use Azurite...** | Checked
**Authorization level** | Function

Click "**Create**".

-----

### Step 2: Code Function

Rename "Function1.cs" to "CustomSkillset.cs". When prompted "You are renaming a file...", click "**Yes**" to perform rename on all references.

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/af1d52cb-5dbd-43b2-9198-ea390dede565" width="800" title="Snipped: October 27, 2023" />

Replace the default logic in "CustomSkillset" with:

```
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using System.Net;
using System.Text.Json;

namespace FunctionApp_CustomSkillset
{
    public class CustomSkillset
    {
        [Function("CustomSkillset")]
        public HttpResponseData Run([HttpTrigger(AuthorizationLevel.Function, "get", "post")] HttpRequestData hrd)
        {
            string request = string.Empty;

            using (var reader = new StreamReader(hrd.Body)) { request = reader.ReadToEnd(); }

            var d = new Dictionary<string, object> { { "values", new List<Dictionary<string, object>>() } };

            foreach (var value in JsonDocument.Parse(request).RootElement.GetProperty("values").EnumerateArray())
            {
                var r = new Dictionary<string, object>
                {
                    { "recordId", value.GetProperty("recordId").GetString() ?? string.Empty },
                    { "data", new Dictionary<string, object> { { "myColumn", value.GetProperty("data") } } },
                    { "errors", string.Empty },
                    { "warnings", string.Empty }
                };

                ((List<Dictionary<string, object>>)d["values"]).Add(r);
            }

            var response = hrd.CreateResponse(HttpStatusCode.OK);
            response.WriteAsJsonAsync(d);
            return response;
        }
    }
}
```

Logic Explained:
* `using (var reader = new StreamReader(hrd.Body)...` parses request body JSON
* `foreach (var value in JsonDocument.Parse(request)` iterates through request values and packages a response
* `response.WriteAsJsonAsync(d)... return...` completes response

_Note: This is a template function ONLY... add additional logic based on your use case_

-----

### Step 3: Publish Function

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/1bed8055-1d94-4c88-bde2-e1c6e557625e" width="800" title="Snipped: October 30, 2023" />

Right-click on your project in the Solution Explorer pane and select "**Publish**" from the resulting menu.

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/5a090114-5879-44d0-a07c-c78a006d92ff" width="600" title="Snipped: October 30, 2023" />

On the "**Publish**" popup, "**Target**" tab, select "**Azure**", then click "**Next**".

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/de97bbd3-ee15-4b1f-af0e-4299f14d6085" width="600" title="Snipped: October 30, 2023" />

On the "**Publish**" popup, "**Specific target**" tab, select "**Azure App Service (Windows)**", then click "**Next**".

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/af141980-d358-4879-b2c9-0277d90882c6" width="600" title="Snipped: October 30, 2023" />

On the **Publish** >> "**Select existing or...**" page, select your Function App and then click "**Finish**".

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/8ed84d8e-1b06-4abf-8cf3-2a158c339d53" width="600" title="Snipped: October 30, 2023" />

On the **Publish profile creation progress** >> "**Finish**" page, click "**Close**".

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/85acb60f-bb87-43a2-9c4a-e4b5f4c12f11" width="800" title="Snipped: October 30, 2023" />

Back on the "...Publish" page, click **Publish**, allow time for processing, and confirm successful publication.

-----

### Step 4: Confirm Success

Navigate to your Azure Function App, then the "**CustomSkillset**" function, and then "**Code + Test**" in the "**Developer**" grouping of the navigation pane.

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/a48bae96-a071-440e-912f-dff11e09da87" width="800" title="Snipped: October 31, 2023" />

Click "**Test/Run**" and on the resulting pop-out, "**Input**" tab, paste the following "**Body**" value:

```
{"values":[{"recordId":"0","data":{"text":"Lorem Ipsum"}}]}
```

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/4d9fd06a-62d8-4ebe-9304-bdcc2834e860" width="800" title="Snipped: October 31, 2023" />

Click "**Run**".

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/7f0f47af-5e58-4fd9-822b-215e910c6602" width="800" title="Snipped: October 31, 2023" />

The pop-out will switch to the "**Output**" tab and you can expect the following "**HTTP response content**" value:

```
{
  "values": [
    {
      "recordId": "0",
      "data": {
        "myColumn": {
          "text": "Lorem Ipsum"
        }
      },
      "errors": "",
      "warnings": ""
    }
  ]
}
```

-----

**Congratulations... you have successfully completed this exercise**

-----
-----

## Exercise 2: Deployment Application
In this exercise, we will use the AI Search Development Kit (SDK) to create a data source, index, skillset, and indexer.

### Step 1: Create Visual Studio Project

Open Visual Studio and click "**Create a new project**".

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/317959b5-dfd7-4c97-af0c-0578f9e89429" width="600" title="Snipped: October 10, 2023" />

On the "**Create a new project**" form, search for and select "**Console App**", then click "**Next**".

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/db8c2898-b607-4441-8b1e-4f4f3dbd56b4" width="600" title="Snipped: October 10, 2023" />

Complete the "**Configure your new project**" form, then click "**Next**".

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/2408d491-ba3b-4ba7-9d84-02caf1dab54d" width="600" title="Snipped: October 10, 2023" />

Complete the "**Additional information**" form, then click "**Create**".

-----

### Step 2: Install NuGet

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/464851d5-30c0-4b72-87d5-cb95658d919d" width="800" title="Snipped: October 11, 2023" />

Click **Tools** in the menu bar, expand "**NuGet Package Manager**" in the resulting menu and then click "**Manage NuGet Packages for Solution...**".

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/a0b3bc3a-e6af-47ff-8ed2-8c0d0340e44e" width="800" title="Snipped: October 11, 2023" />

On the **Browse** tab of the "**NuGet - Solution**" page, search for and select "**Azure.Search.Documents**".
<br>On the resulting pop-out, check the box next to your project and then click "**Install**".

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/8b56a92c-594a-4a18-afaa-24a4872ac73b" width="300" title="Snipped: October 11, 2023" />

When prompted, click "**I Accept**" on the "**License Acceptance**" pop-up.

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/d9906b3b-848d-4807-bc0c-441daf502865" width="800" title="Snipped: October 11, 2023" />

#### Additional Packages

Repeat this process for the following NuGet packages:

* Azure.Identity
* Azure.Security.KeyVault.Secrets

Close the "**NuGet - Solution**" tab.

-----

### Step 3: Code Application

Replace the default code on the "**Program.cs**" tab with the following C#:

```
using Azure;
using Azure.Identity;
using Azure.Search.Documents.Indexes;
using Azure.Search.Documents.Indexes.Models;
using Azure.Security.KeyVault.Secrets;

public class Program
{
    public static void Main(string[] args)
    {

    }
}
```

#### Names, URIs, and Keys
The variables set in this section will be used to identify and create various resources.

Return to the "**Program.cs**" tab and add the following code to `Main`.

```
/* ************************* Names */

string nameBlobStorage_Container = "forms";
string nameAISearch_DataSource = "rchaplerss-datasource";
string nameAISearch_Index = "rchaplerss-index";
string nameAISearch_Indexer = "rchaplerss-indexer";
string nameAISearch_SemanticConfiguration = "rchaplerss-semanticconfiguration";
string nameAISearch_Skillset = "rchaplerss-skillset";
string nameAISearch_Suggester = "rchaplerss-suggester";

/* ************************* URIs */

var uriAISearch = new Uri($"https://rchaplerss.search.windows.net/");
var uriKeyVault = new Uri($"https://rchaplerk.vault.azure.net/");

/* ************************* Keys */

var sc = new SecretClient(uriKeyVault, new DefaultAzureCredential());

var ConnectionString_BlobStorage = sc.GetSecret("ConnectionString-BlobStorage").Value.Value.ToString() ?? string.Empty;
var Key_AISearch = sc.GetSecret("Key-AISearch").Value.Value.ToString() ?? string.Empty;
var Key_AIServices = sc.GetSecret("Key-AIServices").Value.Value.ToString() ?? string.Empty;
/* use of double ".Value" is a necessary oddity */
```

_Notes:_
* _Replace name values {e.g., `rchaplerss`} with values appropriate to your implementation_
* _Replace `AISEARCH_PRIMARYADMINKEY` with your AI Search API Key (and considering using a Key Vault)_

-----

#### Clients
The variables set in this section will be used to manage the AI Search resources.

Append the following code to the bottom of `Main`:

```
/* ************************* Clients */

var credential = new AzureKeyCredential(Key_AISearch);
var indexClient = new SearchIndexClient(uriAISearch, credential);
var indexerClient = new SearchIndexerClient(uriAISearch, credential);
```

Logic Explained:
* `var credential...` creates a new `AzureKeyCredential` object used to authenticate your requests to the AI Search service
* `var indexClient...` creates a new `SearchIndexClient` object used to manage (create, delete, update) indexes in your search service
* `var indexerClient...` creates a new `SearchIndexerClient` object used to manage (run, reset, delete) indexers in your search service

-----

#### Data Source
The logic in this section will create a AI Search Data Source.

Append the following code to the bottom of `Main`:

```
/* ************************* Data Source */

var sidsc = new SearchIndexerDataSourceConnection(
    name: nameAISearch_DataSource,
    type: SearchIndexerDataSourceType.AzureBlob,
    connectionString: ConnectionString_BlobStorage,
    container: new SearchIndexerDataContainer(nameBlobStorage_Container)
    );

indexerClient.DeleteIndexer(nameAISearch_Indexer); /* indexer must be deleted before data source connection */
indexerClient.DeleteDataSourceConnection(nameAISearch_DataSource);

indexerClient.CreateDataSourceConnection(sidsc);
```

Logic Explained:
* `var sidsc...` creates a new `SearchIndexerDataSourceConnection` object that represents a connection to a Blob Storage account
    * Replace `STORAGEACCOUNT_CONNECTIONSTRING` with your Storage Account Connection String (and considering using a Key Vault)
* `indexerClient.CreateDataSourceConnection...` creates a new data source connection using the `SearchIndexerDataSourceConnection` object
* `forms` refers to the Storage Account Container mentioned in Solution Requirements

-----

#### Index
The logic in this section will create a AI Search Index.

Append the following code to the bottom of `Main`:

```
/* ************************* Index */

var index = new SearchIndex(nameAISearch_Index)
{
    Fields = {
        new SimpleField("id", SearchFieldDataType.String) { IsKey = true }, /* SimpleField = non-searchable */
        new SearchField("metadata_author", SearchFieldDataType.String) { IsFacetable = true, IsFilterable = true, IsSortable = true },
        new SearchField("metadata_content_type", SearchFieldDataType.String) { IsFacetable = true, IsFilterable = true, IsSortable = true },
        new SearchField("metadata_creation_date", SearchFieldDataType.String) { IsFacetable = true, IsFilterable = true, IsSortable = true },
        new SearchField("metadata_language", SearchFieldDataType.String) { IsFacetable = true, IsFilterable = true, IsSortable = true },
        new SearchField("metadata_storage_content_type", SearchFieldDataType.String) { IsFacetable = true, IsFilterable = true, IsSortable = true },
        new SearchField("metadata_storage_file_extension", SearchFieldDataType.String) { IsFacetable = true, IsFilterable = true, IsSortable = true },
        new SearchField("metadata_storage_last_modified", SearchFieldDataType.DateTimeOffset) { IsFacetable = true, IsFilterable = true, IsSortable = true },
        new SearchField("metadata_storage_name", SearchFieldDataType.String) { IsFacetable = true, IsFilterable = true, IsSortable = true },
        new SearchField("metadata_storage_path", SearchFieldDataType.String) { IsFacetable = true, IsFilterable = true, IsSortable = true },
        new SearchField("metadata_storage_size", SearchFieldDataType.Int64) { IsFacetable = true, IsFilterable = true, IsSortable = true },
        new SearchField("metadata_title", SearchFieldDataType.String) { IsFacetable = true, IsFilterable = true, IsSortable = true },
        new SearchableField("content") { AnalyzerName = LexicalAnalyzerName.StandardLucene },
        new SearchableField("text", collection: true) { AnalyzerName = LexicalAnalyzerName.StandardLucene },
        new SearchableField("keyphrases", collection: true) { AnalyzerName = LexicalAnalyzerName.StandardLucene },
        new SearchField("myColumn", SearchFieldDataType.String) { IsFacetable = true, IsFilterable = true, IsSortable = true }
    },
    Suggesters = {
        new SearchSuggester(name: nameAISearch_Suggester, sourceFields: new[] { "metadata_storage_name" })
    }
};

indexClient.DeleteIndex(index);
indexClient.CreateIndex(index);
```

Logic Explained:
* `var index...` creates a new `SearchIndex` object that represents a search index in AI Search
  * `new SimpleField(...`, `new SearchField(...`,`new SearchableField(...` lines add fields the index
    * Each field represents a piece of data that can be searched, filtered, sorted, or faceted in the search index
* `new SearchableField("text"...` will be used by the OCR Skill
* `new SearchableField("keyphrases"...` will be used by the Key Phrases Extraction Skill
* `new SearchableField("myColumn"...` will be used by the WebAPI Skill
* `indexClient.DeleteIndex...` deletes any existing index with the same name using the `SearchIndex` object
* `indexClient.CreateIndex...` creates a new index using the `SearchIndex` object

-----

#### Skillset
The logic in this section will create a AI Search Skillset.

Append the following code to the bottom of `Main`:

```
/* ************************* Skillset */

var skills = new List<SearchIndexerSkill>
{
    new OcrSkill(
        inputs: new List<InputFieldMappingEntry>
        {
            new InputFieldMappingEntry("image") { Source = "/document/normalized_images/*" }
        },
        outputs: new List<OutputFieldMappingEntry>
        {
            new OutputFieldMappingEntry("text") { TargetName = "text" }
        }
    )
    {
        Context = "/document/normalized_images/*"
    },
    new KeyPhraseExtractionSkill(
        inputs: new List<InputFieldMappingEntry>
        {
            new InputFieldMappingEntry("text") { Source = "/document/content" }
        },
        outputs: new List<OutputFieldMappingEntry>
        {
            new OutputFieldMappingEntry("keyPhrases") { TargetName = "keyphrases" } /* camel-case required for source */
        }
    )
    {
        Context = "/document/content"
    },
    new WebApiSkill(
        inputs: new List<InputFieldMappingEntry>
        {
            new InputFieldMappingEntry("text") { Source = "/document/metadata_storage_file_extension" }
        },
        outputs: new List<OutputFieldMappingEntry>
        {
            new OutputFieldMappingEntry("myColumn") { TargetName = "myColumn" }
        },
        uri: "{FUNCTIONAPP_URL}"
    )
    {
        Context = "/document/content"
    }
};

var skillset = new SearchIndexerSkillset(nameAISearch_Skillset, skills)
{
    AIServicesAccount = new AIServicesAccountKey(key: Key_AIServices)
};

indexerClient.DeleteSkillset(skillset);
indexerClient.CreateSkillset(skillset);
```

Logic Explained:
* `var skills...` creates a new list of `SearchIndexerSkill` objects
  * Each `SearchIndexerSkill` represents a skill that can be used in an indexing pipeline
  * Inside the `{...}` block, an `OcrSkill` is added to the list
    * The `OcrSkill` is used to extract text from image files
      * The input to the skill is the `image` field, which comes from the `/document/normalized_images/*` path in your data source
      * The output of the skill is the `text` field
    * The `KeyPhraseExtractionSkill` evaluates unstructured text and returns a list of key phrases for each record
      * The input to the skill is `text`, which comes from the `/document/content` path in your data source
      * The output of the skill is the `keyPhrases` field
    * The `WebApiSkill` sends data to a custom web API endpoint (specified by `{FUNCTIONAPP_URL}`) and receives transformed data in return
      * The input is text, sourced from `/document/metadata_storage_file_extension` (used for simplicity)
      * The output from the web API is stored in a field named `myColumn`
* `var skillset...` creates a new `SearchIndexerSkillset` object that represents a skillset in AI Search
  * Inside the `{...}` block, a `AIServicesAccountKey` is set for the skillset
    * This key is used to authenticate your requests to the AI Services
* `indexerClient.DeleteSkillset...` deletes any existing skillset with the same name using the `SearchIndexerSkillset` object
* `indexerClient.CreateSkillset...` creates a new skillset using the `SearchIndexerSkillset` object

-----

#### Indexer
The logic in this section will create a AI Search Indexer.

Append the following code to the bottom of `Main`:

```
/* ************************* Indexer */

var indexer = new SearchIndexer(nameAISearch_Indexer, nameAISearch_DataSource, nameAISearch_Index)
{
    Parameters = new IndexingParameters()
    {
        IndexingParametersConfiguration = new IndexingParametersConfiguration()
        {
            ImageAction = BlobIndexerImageAction.GenerateNormalizedImagePerPage, /* re: OCR */
        }
    },
    SkillsetName = nameAISearch_Skillset
};

indexer.OutputFieldMappings.Add(new FieldMapping(sourceFieldName: "/document/normalized_images/*/text") { TargetFieldName = "text" });
indexer.OutputFieldMappings.Add(new FieldMapping(sourceFieldName: "/document/content/keyphrases") { TargetFieldName = "keyphrases" });
indexer.OutputFieldMappings.Add(new FieldMapping(sourceFieldName: "/document/content/myColumn") { TargetFieldName = "myColumn" });

indexerClient.DeleteIndexer(indexer);
indexerClient.CreateIndexer(indexer);
```

Logic Explained:
* `var indexer...` creates a new SearchIndexer object that represents an indexer in Azure AI Search
  * The indexer is associated with the data source named `dataSourceName` and the index named `indexName`
  * Inside the `{...}` block, parameters are set for the indexer; these parameters control how the indexer behaves during indexing
    * In this case, the `ImageAction` parameter is set to `GenerateNormalizedImagePerPage`, which means the indexer will generate a normalized image for each page of a document
  * Also inside the `{...}` block, the `SkillsetName` property is set to `skillsetName`
    * This means the indexer will use the skillset named `skillsetName` to transform and enrich your data during indexing
* `indexer.OutputFieldMappings...` adds indexer output field mappings
  * `/document/normalized_images/*/text` is mapped to the `text` field in your index
  * `/document/content/keyphrases` is mapped to the `keyphrases` field in your index
  * `/document/content/myColumn` is mapped to the `myColumn` field in your index
* `indexerClient.DeleteIndexer...` deletes any existing indexer with the same name using the `SearchIndexer` object
* `indexerClient.CreateIndexer...` creates a new indexer using the `SearchIndexer` object

-----

### Step 4: Confirm Success

#### Visual Studio Debug

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/94ba39d5-0a55-4439-b2d1-188917087aa4" width="800" title="Snipped: October 11, 2023" />

Save your changes and then click "**Debug**" >> "**Start Debugging**" in the menubar.

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/d00e11c1-5611-49ed-b31e-228eee5428b2" width="600" title="Snipped: October 11, 2023" />

A "Microsoft Visual Studio Debug" window will open (as snipped above).

#### AI Search Index

Navigate to AI Search, then "**Indexes**" in the "**Search management**" grouping of the navigation pane.

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/60745fe0-95f6-40f7-a91b-94b10332c237" width="800" title="Snipped: October 12, 2023" />

You should see the index that you programmatically created {e.g., "rchaplerss-index"}. Click to open.

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/302f8a28-e828-44de-b04c-cb9511bf19a2" width="800" title="Snipped: October 12, 2023" />

Click the "**Search**" button and review results.

-----

**Congratulations... you have successfully completed this exercise**

-----
-----

## Exercise 3: Source Control
In this exercise, we will create a pull request in a DevOps repo.

### Step 1: Create and Push

Open Visual Studio, navigate to "**View**" >> "**Git Changes**", and then click "**Create Git Repository**".

<img src="https://github.com/richchapler/AzureSolutions/assets/44923999/54ffccf9-64db-4e1b-9177-2bde140fa473" width="600" title="Snipped: October 12, 2023" />

Complete the resulting "Create a Git repository" pop-up form, then click "**Create and Push**".

-----

**Congratulations... you have successfully completed all exercises**
