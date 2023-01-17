 
 # Scenario - How to write a query with a serverless SQL pool that will query data from Azure Cosmos DB containers that are enabled with Azure Synapse Link. 
 
 ## Prerequisites
- Make sure that you have prepared Analytical store
- Enable analytical store on your Azure Cosmos DB containers.
- Get the connection string with a read-only key that you will use to query analytical store.
- Get the read-only key that will be used to access the Azure Cosmos DB container


### Sample dataset
The examples in this article are based on data from the [European Centre for Disease Prevention and Control (ECDC) COVID-19 Cases](https://learn.microsoft.com/en-us/azure/open-datasets/dataset-ecdc-covid-cases?tabs=azure-storage) and [COVID-19 Open Research Dataset (CORD-19), doi:10.5281/zenodo.3715505](https://learn.microsoft.com/en-us/azure/open-datasets/dataset-catalog)

### Pre-created Cosmos Container
An Azure Cosmos DB database account that's Azure Synapse Link enabled.
An Azure Cosmos DB database named covid.
Two Azure Cosmos DB containers named Ecdc and Cord19 loaded with the preceding sample datasets.
You can use the following connection string for testing purpose: Account=synapselink-cosmosdb-sqlsample;Database=covid;Key=s5zarR2pT0JWH9k8roipnWxUYBegOuFGjJpSjGlR36y86cW0GQ6RaaG8kGjsRAQoWMw1QKTkkX8HQtFpJjC8Hg==. Note that this connection will not guarantee performance because this account might be located in remote region compared to your Synapse SQL endpoint.

### 1. Explore Azure Cosmos DB data with automatic schema inference

 ### OPENROWSET with key
> SELECT TOP 10 *
FROM OPENROWSET( 
       'CosmosDB',
       'Account=synapselink-cosmosdb-sqlsample;Database=covid;Key=s5zarR2pT0JWH9k8roipnWxUYBegOuFGjJpSjGlR36y86cW0GQ6RaaG8kGjsRAQoWMw1QKTkkX8HQtFpJjC8Hg==',
       Ecdc) as documents
 
 ### OPENROWSET with credential

> /*  Setup - create server-level or database scoped credential with Azure Cosmos DB account key:
    CREATE CREDENTIAL MyCosmosDbAccountCredential
    WITH IDENTITY = 'SHARED ACCESS SIGNATURE', SECRET = 's5zarR2pT0JWH9k8roipnWxUYBegOuFGjJpSjGlR36y86cW0GQ6RaaG8kGjsRAQoWMw1QKTkkX8HQtFpJjC8Hg==';
*/ 
> SELECT TOP 10 *
FROM OPENROWSET(
      PROVIDER = 'CosmosDB',
      CONNECTION = 'Account=synapselink-cosmosdb-sqlsample;Database=covid',
      OBJECT = 'Ecdc',
      SERVER_CREDENTIAL = 'MyCosmosDbAccountCredential'
    ) with ( date_rep varchar(20), cases bigint, geo_id varchar(6) ) as rows
       
We instructed the serverless SQL pool to connect to the covid database in the Azure Cosmos DB account MyCosmosDbAccount authenticated by using the Azure Cosmos DB key (the dummy in the preceding example). We then accessed the Ecdc container's analytical store in the West US 2 region. Since there's no projection of specific properties, the OPENROWSET function will return all properties from the Azure Cosmos DB items.


### 2. Explore data from the other container in the same Azure Cosmos DB database, you can use the same connection string and reference the required container as the third parameter:

> SELECT TOP 10 *
FROM OPENROWSET( 
       'CosmosDB',
       'Account=synapselink-cosmosdb-sqlsample;Database=covid;Key=s5zarR2pT0JWH9k8roipnWxUYBegOuFGjJpSjGlR36y86cW0GQ6RaaG8kGjsRAQoWMw1QKTkkX8HQtFpJjC8Hg==',
       Cord19) as cord19
       
  


 
 
