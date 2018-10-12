# Lab 1: Create a Cognitive Search Enrichment Process with **Text** Skills

In this lab, you will learn the mechanics of programming data enrichment in Azure Search using *cognitive skills*. Cognitive skills are natural language processing (NLP) and image analysis operations that extract text and text representations of an image, detect language, entities, key phrases, and more. The end result is rich additional content in an Azure Search index, created by a cognitive search indexing pipeline. 

**All the links in this lab are extra content for your learning, but you don't need them to perform the required activities.**

In this lab, we will learn how to create a Cognitive Search indexing pipeline that enriches source data in route to an index. The output is a full text searchable index on Azure Search. 


The list of activities we will do, using Azure Search REST APIs, is:

+ Create a data source for the uploaded data.
+ Create a Cognitive Search Skillset with entity recognition, language detection, text manipulation and key phrase extraction.
+ Create an index to store the enriched metadata.
+ Create an indexer process to execute the enrichment.
+ Check the indexer status
+ Check the enriched metadata
+ Query the metadata


>TIP for Later: You can enhance the index with other Azure Search standard capabilities, such as [synonyms](https://docs.microsoft.com/en-us/azure/search/search-synonyms), [scoring profiles](https://docs.microsoft.com/rest/api/searchservice/add-scoring-profiles-to-a-search-index), [analyzers](https://docs.microsoft.com/en-us/rest/api/searchservice/custom-analyzers-in-azure-search), and [filters](https://docs.microsoft.com/en-us/azure/search/search-filters).

## Step 1 - Create a data source

Now that your services and source files are prepared, start assembling the components of your indexing pipeline. We'll begin by creating a [data source object](https://docs.microsoft.com/rest/api/searchservice/create-data-source) that tells Azure Search how to retrieve external source data.

For this tutorial, we will use Postman to call Azure Search service APIs. In the request header, provide the service name you used while creating the Azure Search service, and the api-key generated for your search service. In the request body, specify the blob container name and connection string.

### Sample Request
```http
POST https://[service name].search.windows.net/datasources?api-version=2017-11-11-Preview  
api-key: [admin key]  
Content-Type: application/json
```
#### Request Body Syntax
```json
{   
    "name" : "demodata",  
    "description" : "Demo files to demonstrate cognitive search capabilities.",  
    "type" : "azureblob",
    "credentials" :
    { "connectionString" :
      "DefaultEndpointsProtocol=https;AccountName=<your account name>;AccountKey=<your account key>;"
    },  
    "container" : { "name" : "<your blob container name>" }
}  
```
Send the request. The web test tool should return a status code of 201 confirming success. 

Since this is your first request, check the Azure portal to confirm the data source was created in Azure Search. On the search service dashboard page, verify the Data Sources tile has a new item. You might need to wait a few minutes for the portal page to refresh. 

  ![Data sources tile in the portal](./media/data-source.PNG "Data sources tile in the portal")

If you got a 403 or 404 error, check the request construction: `api-version=2017-11-11-Preview` should be on the endpoint, `api-key` should be in the Header after `Content-Type`, and its value must be valid for a search service. You can reuse the header for the remaining steps in this lab.

> [!TIP]
> Now, before you do any more work and you're in the portal anyway, is a good time to verify that the search service is running in one of the supported locations providing the preview feature: South Central US or West Europe.

## Step 2 - Create a skillset

In this step, you define a set of enrichment steps that you want to apply to your data. Each enrichment step is called a *skill*, and the set of enrichment steps a *skillset*. This tutorial uses the following [predefined cognitive skills](https://docs.microsoft.com/en-us/azure/search/cognitive-search-predefined-skills):

+ [Language Detection](https://docs.microsoft.com/en-us/azure/search/cognitive-search-skill-language-detection) to identify the content's language.

+ [Text Split](https://docs.microsoft.com/en-us/azure/search/cognitive-search-skill-textsplit) to break large content into smaller chunks before calling the key phrase extraction skill. Key phrase extraction accepts inputs of 50,000 characters or less. A few of the sample files need splitting up to fit within this limit.

+ [Named Entity Recognition](https://docs.microsoft.com/en-us/azure/search/cognitive-search-skill-named-entity-recognition) for extracting the names of organizations from content in the blob container.

+ [Key Phrase Extraction](https://docs.microsoft.com/en-us/azure/search/cognitive-search-skill-keyphrases) to pull out the top key phrases. 

### Sample Request
Before you make this REST call, remember to replace the service name and the admin key in the request below if your tool does not preserve the request header between calls. 

This request creates a skillset. Reference the skillset name ```demoskillset``` for the rest of this lab.

```http
PUT https://[servicename].search.windows.net/skillsets/demoskillset?api-version=2017-11-11-Preview
api-key: [admin key]
Content-Type: application/json
```
#### Request Body Syntax
```json
{
  "description": 
  "Extract entities, detect language and extract key-phrases",
  "skills":
  [
    {
      "@odata.type": "#Microsoft.Skills.Text.NamedEntityRecognitionSkill",
      "categories": [ "Organization" ],
      "defaultLanguageCode": "en",
      "inputs": [
        {
          "name": "text", "source": "/document/content"
        }
      ],
      "outputs": [
        {
          "name": "organizations", "targetName": "organizations"
        }
      ]
    },
    {
      "@odata.type": "#Microsoft.Skills.Text.LanguageDetectionSkill",
      "inputs": [
        {
          "name": "text", "source": "/document/content"
        }
      ],
      "outputs": [
        {
          "name": "languageCode",
          "targetName": "languageCode"
        }
      ]
    },
    {
      "@odata.type": "#Microsoft.Skills.Text.SplitSkill",
      "textSplitMode" : "pages", 
      "maximumPageLength": 4000,
      "inputs": [
      {
        "name": "text",
        "source": "/document/content"
      },
      { 
        "name": "languageCode",
        "source": "/document/languageCode"
      }
    ],
    "outputs": [
      {
            "name": "textItems",
            "targetName": "pages"
      }
    ]
  },
  {
      "@odata.type": "#Microsoft.Skills.Text.KeyPhraseExtractionSkill",
      "context": "/document/pages/*",
      "inputs": [
        {
          "name": "text", "source": "/document/pages/*"
        },
        {
          "name":"languageCode", "source": "/document/languageCode"
        }
      ],
      "outputs": [
        {
          "name": "keyPhrases",
          "targetName": "keyPhrases"
        }
      ]
    }
  ]
}
```

Send the request. The web test tool should return a status code of 201 confirming success. 

#### About the request

Let's take some time to review the request and confirm we understand what's happening and how.  

Notice how the key phrase extraction skill is applied for each page. By setting the context to ```"document/pages/*"``` you run this enricher for each member of the document/pages array (for each page in the document).

Each skill executes on the content of the document. During processing, Azure Search cracks each document to read content from different file formats. Found text originating in the source file is placed into a generated ```content``` field, one for each document. As such, set the input as ```"/document/content"```.

A graphical representation of the skillset you created is shown below. 

![Understand a skillset](./media/skillset.png "Understand a skillset")

Outputs can be mapped to an index, used as input to a downstream skill, or both as is the case with language code. In the index, a language code is useful for filtering. As an input, language code is used by text analysis skills to inform the linguistic rules around word breaking.

For more information about skillset fundamentals, read [how to define a skillset](cognitive-search-defining-skillset.md).

## Step 3 - Create an index

In this section, you define the index schema by specifying which fields to include in the searchable index, and the search attributes for each field. Fields have a type and can take attributes that determine how the field is used (searchable, sortable, and so forth). Field names in an index are not required to identically match the field names in the source. In a later step, you add field mappings in an indexer to connect source-destination fields. For this step, define the index using field naming conventions pertinent to your search application.

This exercise uses the following fields and field types:

| field-names: | id       | content   | languageCode | keyPhrases         | organizations     |
|--------------|----------|-------|----------|--------------------|-------------------|
| field-types: | Edm.String|Edm.String| Edm.String| List<Edm.String>  | List<Edm.String>  |


### Sample Request
Before you make this REST call, remember to replace the service name and the admin key in the request below if your tool does not preserve the request header between calls. 

This request creates an index. Use the index name ```demoindex``` for the rest of this tutorial.

```http
PUT https://[servicename].search.windows.net/indexes/demoindex?api-version=2017-11-11-Preview
api-key: [api-key]
Content-Type: application/json
```
#### Request Body Syntax

```json
{
  "fields": [
    {
      "name": "id",
      "type": "Edm.String",
      "key": true,
      "searchable": true,
      "filterable": false,
      "facetable": false,
      "sortable": true
    },
    {
      "name": "content",
      "type": "Edm.String",
      "sortable": false,
      "searchable": true,
      "filterable": false,
      "facetable": false
    },
    {
      "name": "languageCode",
      "type": "Edm.String",
      "searchable": true,
      "filterable": false,
      "facetable": false
    },
    {
      "name": "keyPhrases",
      "type": "Collection(Edm.String)",
      "searchable": true,
      "filterable": false,
      "facetable": false
    },
    {
      "name": "organizations",
      "type": "Collection(Edm.String)",
      "searchable": true,
      "sortable": false,
      "filterable": false,
      "facetable": false
    }
  ]
}
```
Send the request. The web test tool should return a status code of 201 confirming success. 

Review the request and confirm understanding. If you want to learn more about defining an index, see [Create Index (Azure Search REST API)](https://docs.microsoft.com/rest/api/searchservice/create-index).


## Step 4 - Create an indexer, map fields, and execute transformations

So far you have created a data source, a skillset, and an index. These three components become part of an [indexer](https://docs.microsoft.com/en-us/azure/search/search-indexer-overview) that pulls each piece together into a single multi-phased operation. To tie these together in an indexer, you must define field mappings. Field mappings are part of the indexer definition and execute the transformations when you submit the request.

For non-enriched indexing, the indexer definition provides an optional *fieldMappings* section if field names or data types do not precisely match, or if you want to use a function.

For cognitive search workloads having an enrichment pipeline, an indexer requires *outputFieldMappings*. These mappings are used when an internal process (the enrichment pipeline) is the source of field values. Behaviors unique to *outputFieldMappings* include the ability to handle complex types created as part of enrichment (through the shaper skill). Also, there may be many elements per document (for instance, multiple organizations in a document). The *outputFieldMappings* construct can direct the system to "flatten" collections of elements into a single record.

**The next step takes up to 10 minutes of processing to complete.**

### Sample Request

Before you make this REST call, remember to replace the service name and the admin key in the request below if your tool does not preserve the request header between calls. 

Also, provide the name of your indexer. You can reference it as ```demoindexer``` for the rest of this lab.

```http
PUT https://[servicename].search.windows.net/indexers/demoindexer?api-version=2017-11-11-Preview
api-key: [api-key]
Content-Type: application/json
```
#### Request Body Syntax

```json
{
  "name":"demoindexer",	
  "dataSourceName" : "demodata",
  "targetIndexName" : "demoindex",
  "skillsetName" : "demoskillset",
  "fieldMappings" : [
        {
          "sourceFieldName" : "metadata_storage_path",
          "targetFieldName" : "id",
          "mappingFunction" : 
            { "name" : "base64Encode" }
        },
        {
          "sourceFieldName" : "content",
          "targetFieldName" : "content"
        }
   ],
  "outputFieldMappings" : 
  [
        {
          "sourceFieldName" : "/document/organizations", 
          "targetFieldName" : "organizations"
        },
        {
          "sourceFieldName" : "/document/pages/*/keyPhrases/*", 
          "targetFieldName" : "keyPhrases"
        },
        {
            "sourceFieldName": "/document/languageCode",
            "targetFieldName": "languageCode"
        }      
  ],
  "parameters":
  {
  	"maxFailedItems":-1,
  	"maxFailedItemsPerBatch":-1,
  	"configuration": 
    {
    	"dataToExtract": "contentAndMetadata",
     	"imageAction": "generateNormalizedImages"
		}
  }
}
```

Send the request. The web test tool should return a status code of 201 confirming successful processing. 

Expect this step to take several minutes to complete. Even though the data set is small, analytical skills are computation-intensive. Some skills, such as image analysis, are long-running.

> [!TIP]
> Creating an indexer invokes the pipeline. If there are problems reaching the data, mapping inputs and outputs, or order of operations, they appear at this stage. To re-run the pipeline with code or script changes, you might need to drop objects first. For more information, see [Reset and re-run](https://docs.microsoft.com/en-us/azure/search/cognitive-search-tutorial-blob#reset).

### Explore the request body

The script sets ```"maxFailedItems"```  to -1, which instructs the indexing engine to ignore errors during data import. This is useful because there are so few documents in the demo data source. For a larger data source, you would set the value to greater than 0.

Also notice the ```"dataToExtract":"contentAndMetadata"``` statement in the configuration parameters. This statement tells the indexer to automatically extract the content from different file formats as well as metadata related to each file. 

When content is extracted, you can set ```ImageAction``` to extract text from images found in the data source. The ```"ImageAction":"generateNormalizedImages"``` tells the indexer to extract text from the images (for example, the word "stop" from a traffic Stop sign), and embed it as part of the content field. This behavior applies to both the images embedded in the documents (think of an image inside a PDF), as well as images found in the data source, for instance a JPG file.

In this preview, ```"generateNormalizedImages"``` is the only valid value for ```"ImageAction"```.

## Step 5 - Check indexer status

Once the indexer is defined, it runs automatically when you submit the request. Depending on which cognitive skills you defined, indexing can take longer than you expect. To find out whether the indexer is still running, send the following request to check the indexer status.

```http
GET https://[servicename].search.windows.net/indexers/demoindexer/status?api-version=2017-11-11-Preview
api-key: [api-key]
Content-Type: application/json
```

The response tells you whether the indexer is running. Once indexing is finished, the response to the same call (as above) will result in a report of any errors and warnings that occurred during enrichment.  

Warnings are common with some source file and skill combinations and do not always indicate a problem. In this lab, the warnings are benign (for example, no text inputs from the JPEG files). You can review the status response for verbose information about warnings emitted during indexing. You can also expect warnings about text been truncated or long words. **If the status field has "success", we don't need to worry about any of the warnings.**

## Step 6 - Verify content

After indexing is finished, run queries that return the contents of individual fields. By default, Azure Search returns the top 50 results. The sample data is small so the default works fine. However, when working with larger data sets, you might need to include parameters in the query string to return more results - you can read [how to page results in Azure Search here](https://docs.microsoft.com/en-us/azure/search/search-pagination-page-layout).

As a verification step, query the index for all of the fields.

```http
GET https://[servicename].search.windows.net/indexes/demoindex?api-version=2017-11-11-Preview
api-key: [api-key]
Content-Type: application/json
```

The output is the index schema, with the name, type, and attributes of each field.

Submit a second query for `"*"` to return all contents of a single field, such as `organizations`.

```http
GET https://[servicename].search.windows.net/indexes/demoindex/docs?search=*&$select=organizations&api-version=2017-11-11-Preview
api-key: [api-key]
Content-Type: application/json
```

Repeat for additional fields: content, languageCode, keyPhrases, and organizations in this exercise. Fiedls names in your query must match with the names you defined in your index. And it is case sensitive. The output of the query above helps you to confirm the index fields names.  You can return multiple fields via `$select` using a comma-delimited list. 

You can use GET or POST, depending on query string complexity and length. For more information, see [Query using the REST API](https://docs.microsoft.com/azure/search/search-query-rest-api).

<a name="access-enriched-document"></a>

## Next Step
[Lab 2](05-Lab-2-Image-Skills.md) or
[Back to Main Menu](readme.md)

