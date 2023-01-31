# Surface Data with Bulk Export

![image](https://user-images.githubusercontent.com/44923999/215880260-d581ec97-3b29-4490-94ae-0f473fcc3612.png)

## Use Case
This solution considers the following requirements:

* "We have hundreds of Data Explorer tables that we want to ready for use with Cognitive Search"
* "Individual data export will take much more human effort than we want to invest"

## Required Infrastructure
This solution requires the following resources:

* [**Data Explorer**](https://learn.microsoft.com/en-us/azure/data-explorer/)
  * [**Cluster**](Infrastructure_DataExplorer_Cluster.md)
  * [**Database**](Infrastructure_DataExplorer_Database.md) with **Viewer** permissions for Synapse managed identity
  * Three copies of [sample data](https://learn.microsoft.com/en-us/azure/data-explorer/ingest-sample-data?tabs=ingestion-wizard) {i.e., tables StormEvents1, StormEvents2, and StormEvents3}
* [**Storage Account**](Infrastructure_StorageAccount.md) with container "exports" (and related SAS token)
* [**Synapse**](Infrastructure_Synapse.md) with your Client IP added to firewall

## Proposed Solution
This solution will address requirements in N exercises:

* Exercise 1: Synapse Pipeline, Create Export Iterator

-----

## Exercise 1: Data Exploer, Export Data
In this exercise, we will explore the identification of export targets and how to automate export.

![image](https://user-images.githubusercontent.com/44923999/215788562-676e145f-5baf-4843-a74f-7256b341f53e.png)

### Step 1: Perform One-Time Export (Option #1)
In this step, we will: 1) load sample data and 2) run a KQL query to perform an export to Blob Storage.