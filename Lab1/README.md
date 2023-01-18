 
 # Scenario - How to write a query with a serverless SQL pool that will query data from Azure Cosmos DB containers that are enabled with Azure Synapse Link. 
 ![Cosmos SQL Mapping](/Lab1/images/adhocanalyticsscenario.jpg)
 
 ## Prerequisites
- Make sure that you have prepared Analytical store
- Enable analytical store on your Azure Cosmos DB containers.
- Get the connection string with a read-only key that you will use to query analytical store.
- Get the read-only key that will be used to access the Azure Cosmos DB container
- Create [Azure Synapse workspace](https://docs.microsoft.com/azure/synapse-analytics/quickstart-create-workspace) in West US 2


### Pre-created Cosmos Container
An Azure Cosmos DB database account that's Azure Synapse Link enabled.
An Azure Cosmos DB database named covid.
Two Azure Cosmos DB containers named Ecdc and Cord19 loaded with the preceding sample datasets.
You can use the following connection string for testing purpose: Account=synapselink-cosmosdb-sqlsample;Database=covid;Key=s5zarR2pT0JWH9k8roipnWxUYBegOuFGjJpSjGlR36y86cW0GQ6RaaG8kGjsRAQoWMw1QKTkkX8HQtFpJjC8Hg==. Note that this connection will not guarantee performance because this account might be located in remote region compared to your Synapse SQL endpoint.
### Sample dataset (Reference)
The examples in this article are based on data from the [European Centre for Disease Prevention and Control (ECDC) COVID-19 Cases](https://learn.microsoft.com/en-us/azure/open-datasets/dataset-ecdc-covid-cases?tabs=azure-storage) and [COVID-19 Open Research Dataset (CORD-19), doi:10.5281/zenodo.3715505](https://learn.microsoft.com/en-us/azure/open-datasets/dataset-catalog)

### 1. Explore Azure Cosmos DB data with automatic schema inference

 ### OPENROWSET with key
 
``` sql
 SELECT TOP 10 *
FROM OPENROWSET( 
       'CosmosDB',
       'Account=synapselink-cosmosdb-sqlsample;Database=covid;Key=s5zarR2pT0JWH9k8roipnWxUYBegOuFGjJpSjGlR36y86cW0GQ6RaaG8kGjsRAQoWMw1QKTkkX8HQtFpJjC8Hg==',
       Ecdc) as documents
 ``` 
 
 ### OPENROWSET with credential
  Setup - create server-level or database scoped credential with Azure Cosmos DB account key:

``` sql
    CREATE CREDENTIAL MyCosmosDbAccountCredential
    WITH IDENTITY = 'SHARED ACCESS SIGNATURE', SECRET = 's5zarR2pT0JWH9k8roipnWxUYBegOuFGjJpSjGlR36y86cW0GQ6RaaG8kGjsRAQoWMw1QKTkkX8HQtFpJjC8Hg==';
```


``` sql
  SELECT TOP 10 *
FROM OPENROWSET(
      PROVIDER = 'CosmosDB',
      CONNECTION = 'Account=synapselink-cosmosdb-sqlsample;Database=covid',
      OBJECT = 'Ecdc',
      SERVER_CREDENTIAL = 'MyCosmosDbAccountCredential'
    ) with ( date_rep varchar(20), cases bigint, geo_id varchar(6) ) as rows
    
``` 
       
We instructed the serverless SQL pool to connect to the covid database in the Azure Cosmos DB account MyCosmosDbAccount authenticated by using the Azure Cosmos DB key (the dummy in the preceding example). We then accessed the Ecdc container's analytical store in the West US 2 region. Since there's no projection of specific properties, the OPENROWSET function will return all properties from the Azure Cosmos DB items.

# Azure Cosmos DB to SQL type mappings
Although Azure Cosmos DB transactional store is schema-agnostic, the analytical store is schematized to optimize for analytical query performance.
The following table shows the SQL column types that should be used for different property types in Azure Cosmos DB.
![Cosmos SQL Mapping](/Lab1/images/CosmosDB_SQLtypemappings.jpg)



### Explore data from the other container in the same Azure Cosmos DB database, you can use the same connection string and reference the required container as the third parameter:
 ### OPENROWSET with key


``` sql
SELECT TOP 10 *
FROM OPENROWSET( 
       'CosmosDB',
       'Account=synapselink-cosmosdb-sqlsample;Database=covid;Key=s5zarR2pT0JWH9k8roipnWxUYBegOuFGjJpSjGlR36y86cW0GQ6RaaG8kGjsRAQoWMw1QKTkkX8HQtFpJjC8Hg==',
       Cord19) as cord19
       
```

### 2. Explicitly specify schema

The data from the ECDC COVID dataset is in following structure into Azure Cosmos DB:

 ```yaml
{"date_rep":"2020-08-13","cases":254,"countries_and_territories":"Serbia","geo_id":"RS"}
{"date_rep":"2020-08-12","cases":235,"countries_and_territories":"Serbia","geo_id":"RS"}
{"date_rep":"2020-08-11","cases":163,"countries_and_territories":"Serbia","geo_id":"RS"}
```



 ### OPENROWSET with key

``` sql
SELECT TOP 10 *
FROM OPENROWSET(
      'CosmosDB',
      'Account=synapselink-cosmosdb-sqlsample;Database=covid;Key=s5zarR2pT0JWH9k8roipnWxUYBegOuFGjJpSjGlR36y86cW0GQ6RaaG8kGjsRAQoWMw1QKTkkX8HQtFpJjC8Hg==',
       Ecdc
    ) with ( date_rep varchar(20), cases bigint, geo_id varchar(20) ) as rows
 
``` 

 ### OPENROWSET with credential

  Setup - create server-level or database scoped credential with Azure Cosmos DB account key: (If not created in previous step)


``` sql
    CREATE CREDENTIAL MyCosmosDbAccountCredential
    WITH IDENTITY = 'SHARED ACCESS SIGNATURE', SECRET = 's5zarR2pT0JWH9k8roipnWxUYBegOuFGjJpSjGlR36y86cW0GQ6RaaG8kGjsRAQoWMw1QKTkkX8HQtFpJjC8Hg==';

```


``` sql
SELECT TOP 10 *
FROM OPENROWSET(
      PROVIDER = 'CosmosDB',
      CONNECTION = 'Account=synapselink-cosmosdb-sqlsample;Database=covid',
      OBJECT = 'Ecdc',
      SERVER_CREDENTIAL = 'MyCosmosDbAccountCredential'
    ) with ( date_rep varchar(20), cases bigint, geo_id varchar(20) ) as rows
  
```  
 
    
 ### 3. Creating view on top of your Azure Cosmos DB data
  
``` sql
CREATE OR ALTER VIEW Ecdc
AS SELECT *
FROM OPENROWSET(
      PROVIDER = 'CosmosDB',
      CONNECTION = 'Account=synapselink-cosmosdb-sqlsample;Database=covid',
      OBJECT = 'Ecdc',
      SERVER_CREDENTIAL = 'MyCosmosDbAccountCredential'
    ) with ( date_rep varchar(20), cases bigint, geo_id varchar(20) ) as rows
  
  
``` 
 
 Cross check if the view is created sucessfully or not.
 
 
``` sql
 SELECT TOP (100) [date_rep],[cases],[geo_id] FROM [dbo].[Ecdc]
 
``` 
    
 ### 4. How to query nested objects
  CORD-19 dataset has JSON documents that follow this structure:
  ```yaml
  {
    "paper_id": <str>,                   # 40-character sha1 of the PDF
    "metadata": {
        "title": <str>,
        "authors": <array of objects>    # list of author dicts, in order
        ...
     }
     ...
}
```
You can specify the paths to nested values in the objects when you use the WITH clause.

``` sql
SELECT TOP 10 *
FROM OPENROWSET( 
       'CosmosDB',
       'Account=synapselink-cosmosdb-sqlsample;Database=covid;Key=s5zarR2pT0JWH9k8roipnWxUYBegOuFGjJpSjGlR36y86cW0GQ6RaaG8kGjsRAQoWMw1QKTkkX8HQtFpJjC8Hg==',
       Cord19)
WITH (  paper_id    varchar(8000),
        title        varchar(2000) '$.metadata.title',
        metadata     varchar(max),
        authors      varchar(max) '$.metadata.authors'
) AS docs;

``` 

 ### 5. How to flatten nested arrays
  Data might have nested subarrays like the author's array from a CORD-19 dataset
```yaml
  {
    "paper_id": <str>,                      # 40-character sha1 of the PDF
    "metadata": {
        "title": <str>,
        "authors": [                        # list of author dicts, in order
            {
                "first": <str>,
                "middle": <list of str>,
                "last": <str>,
                "suffix": <str>,
                "affiliation": <dict>,
                "email": <str>
            },
            ...
        ],
        ...
}
```

 ``` sql
 SELECT
    *
FROM
    OPENROWSET(
      'CosmosDB',
      'Account=synapselink-cosmosdb-sqlsample;Database=covid;Key=s5zarR2pT0JWH9k8roipnWxUYBegOuFGjJpSjGlR36y86cW0GQ6RaaG8kGjsRAQoWMw1QKTkkX8HQtFpJjC8Hg==',
       Cord19
    ) WITH ( title varchar(2000) '$.metadata.title',
             authors varchar(max) '$.metadata.authors' ) AS docs
      CROSS APPLY OPENJSON ( authors )
                  WITH (
                       first varchar(50),
                       last varchar(50),
                       affiliation nvarchar(max) as json
                  ) AS a
 
 ```
  A serverless SQL pool enables you to flatten nested structures by applying the OPENJSON function on the nested array
       

       
  


 
 
