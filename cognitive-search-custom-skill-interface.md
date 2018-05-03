#Lab 1.2 -  Custom Skills interface definition

Azure Search includes a range of [built-in skills](cognitive-search-predefined-skills.md) that can be used for [enrichment and augmentation](cognitive-search-concept-intro.md) in an indexing pipeline. 

Additionally, you can create custom skills to insert transformations unique to your content. A custom skill executes independently, applying whatever augmentation or enrichment step you require. For example, you could define field-specific custom entities, build custom classification models to differentiate business and financial contracts and documents, or add a speech recognition skill to reach deeper into audio files for relevant content. For a step-by-step example, see [Example: creating a custom skill](cognitive-search-create-custom-skill-example.md).

 Whatever custom capability you might require, there is a simple and clear interface for connecting a custom skill to the rest of the enrichment pipeline. The only requirement for inclusion in a [skillset](cognitive-search-defining-skillset.md) is the ability to accept inputs and emit outputs in ways that are consumable within the skillset as a whole. The focus of this article is on the input and output formats that the enrichment pipeline requires.

## Web API custom skill interface

Currently, the only mechanism for interacting with a custom skill is through a Web API interface. The Web API needs must meet the requirements described in this section.

### 1.  Web API Input Format

The Web API must accept an array of records to be processed. Each record must contain a "property bag" that will be the input provided to your Web API. 

Suppose that you want to create a simple enricher that identifies the first date mentioned in the text of a contract. In this example, the skill accepts a single input *contractText* as the contract text. The skill also has a single output, which is the date of the contract. To make the enricher more interesting, return this *contractDate* in the shape of a multi-part complex type.

Your Web API should be ready to receive a batch of input records. Each member of the *values* array represents the input for a particular record. Each record is required to have the following elements:

+ a *recordId* member that is the unique identifier for a particular record. When your enricher returns the results, it must provide this *recordId* in order to allow the caller to match the record results to their input.

+ a *data* member, which is essentially a bag of input fields for each record.

To be more concrete, for our example above, your Web API should expect requests that look like this:

```json
{
    "values": [
      {
        "recordId": "a1",
        "data":
           {
             "contractText": 
                "This is a contract that was issues on November 3, 2017 and that involves... "
           }
      },
      {
        "recordId": "b5",
        "data":
           {
             "contractText": 
                "In the City of Seattle, WA on February 5, 2018 there was a decision made..."
           }
      },
      {
        "recordId": "c3",
        "data":
           {
             "contractText": null
           }
      }
    ]
}
```
In reality, your service may get called with hundreds or thousands of records instead of only the three shown here.

### 2. Web API Output Format

The format of the output is a set of records containing a *recordId*, and a property bag 

```json
{
  "values": 
  [
      {
        "recordId": "b5",
        "data" : 
        {
            "contractDate":  { "day" : 5, "month": 2, "year" : 2018 }
        }
      },
      {
        "recordId": "a1",
        "data" : {
            "contractDate": { "day" : 3, "month": 11, "year" : 2017 }                    
        }
      },
      {
        "recordId": "c3",
        "data" : 
        {
        },
        "errors": [ { "message": "contractText field required "}   ],  
        "warnings": [ {"message": "Date not found" }  ]
      }
    ]
}
```

This particular example has only one output, but you could output more than one property. 

### Errors and Warning

As the previous example shows, you may return error and warning messages for each record. 

## Consuming custom skills from skillset

When you create a Web API enricher, you can describe HTTP headers and parameters as part of the request. The snippet below shows how request parameters and HTTP headers may be described as part of the skillset definition.

```json
{
    "skills": [
      {
        "@odata.type": "#Microsoft.Skills.Custom.WebApiSkill",
        "description": "This skill calls an Azure function, which in turn calls TA sentiment",
        "uri": "https://indexer-e2e-webskill.azurewebsites.net/api/DateExtractor?language=en",
        "context": "/document",
        "httpHeaders": {
            "DateExtractor-Api-Key": "foo"
        },
        "inputs": [
          {
            "name": "contractText",
            "source": "/document/content"
          }
        ],
        "outputs": [
          {
            "name": "contractDate",
            "targetName": "date"
          }
        ]
      }
  ]
}
```

## See also
+ [How to define a skillset](cognitive-search-defining-skillset.md)
+ [Create Skillset (REST)](ref-create-skillset.md)
+ [LAB 3 - Mapping enriched fields](cognitive-search-output-field-mapping.md)

## Next Step
[Back to Labs Main Menu](readme.md)
