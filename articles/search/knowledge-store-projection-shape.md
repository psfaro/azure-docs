---
title: Shaping data for knowledge store 
titleSuffix: Azure Cognitive Search
description: Define the data structures in a knowledge store by creating data shapes and passing them to a projection.

author: HeidiSteen
ms.author: heidist
ms.service: cognitive-search
ms.topic: conceptual
ms.date: 08/10/2021
---

# Shaping data for projection into a knowledge store

In Azure Cognitive Search, "shaping data" describes a step in the [knowledge store workflow](knowledge-store-concept-intro.md) that creates a data representation of the content that you want to project into tables, objects, and files in Azure Storage.

As skills execute, the outputs are written to an enrichment tree in a hierarchy of nodes, and while you might want to view and consume the enrichment tree in its entirety, it's more likely that you will want a finer grain, creating subsets of nodes for different scenarios, such as placing the nodes related to translated text or extracted entities in specific tables.

By itself, the enrichment tree does not include logic that would inform how its content is represented in a knowledge store. Data shapes fill this gap by providing the schema of what goes into each table, object, and file projection. You can think of a data shape as a custom definition or view of the enriched data. You can create as many shapes as you need, and then assign them to projections in a knowledge store definition. 

## Approaches for creating shapes

There are two ways to shape enriched content to that it can be projected into a knowledge store:

+ Use the [Shaper skill](cognitive-search-skill-shaper.md) to create nodes in an enrichment tree that are used expressly for projection. Most skills create new content. In contrast, a Shaper skill work with existing nodes, usually to consolidate multiple nodes into a single complex object. This is useful for tables, where you want the output of multiple nodes to be physically expressed as columns in the table. 

+ Use an inline shape within the projection definition itself.

Using the Shaper skill externalizes the shape so that it can be used by multiple projections or even other skills. It also ensures that all the mutations of the enrichment tree are contained within the skill, and that the output is an object that can be reused. In contrast, inline shaping allows you to create the shape you need, but is an anonymous object and is only available to the projection for which it is defined.

The approaches can be used together or separately. This article shows both: a Shaper skill for the table projections, and inline shaping with the key phrases table projection.

## Use a Shaper skill

Shaper skills are usually placed at the end of a skillset, creating a view of the data that you want to pass to a projection. This example creates a shape called "tableprojection" containing the following nodes: "reviews_text", "reviews_title", "AzureSearch_DocumentKey", and sentiment scores and key phrases from paged reviews. 

```json
{
    "@odata.type": "#Microsoft.Skills.Util.ShaperSkill",
    "name": "#5",
    "description": null,
    "context": "/document",
    "inputs": [        
        {
            "name": "reviews_text",
            "source": "/document/reviews_text",
            "sourceContext": null,
            "inputs": []
        },
        {
            "name": "reviews_title",
            "source": "/document/reviews_title",
            "sourceContext": null,
            "inputs": []
        },
        {
            "name": "AzureSearch_DocumentKey",
            "source": "/document/AzureSearch_DocumentKey",
            "sourceContext": null,
            "inputs": []
        },  
        {
            "name": "pages",
            "source": null,
            "sourceContext": "/document/reviews_text/pages/*",
            "inputs": [
                {
                    "name": "Sentiment",
                    "source": "/document/reviews_text/pages/*/Sentiment",
                    "sourceContext": null,
                    "inputs": []
                },
                {
                    "name": "LanguageCode",
                    "source": "/document/Language",
                    "sourceContext": null,
                    "inputs": []
                },
                {
                    "name": "Page",
                    "source": "/document/reviews_text/pages/*",
                    "sourceContext": null,
                    "inputs": []
                },
                {
                    "name": "keyphrase",
                    "sourceContext": "/document/reviews_text/pages/*/Keyphrases/*",
                    "inputs": [
                        {
                            "source": "/document/reviews_text/pages/*/Keyphrases/*",
                            "name": "Keyphrases"
                        }
                    ]
                }
            ]
        }
    ],
    "outputs": [
        {
            "name": "output",
            "targetName": "tableprojection"
        }
    ]
}
```

### SourceContext property

Within a Shaper skill, an input can have a `sourceContext` element. This same property can also be used in inline shapes in projections. 

`sourceContext` is used to construct multi-level, nested objects in an enrichment pipeline. If the input is at a *different* context than the skill context, use the *sourceContext*. The *sourceContext* requires you to define a nested input with the specific element being addressed as the source. 

In the example above, sentiment analysis and key phrases extraction was performed on text that was split into pages for more efficient analysis. Assuming you want the scores and phrases projected into a table, you will now need to set the context to nested input that provides the score and phrase.

### Projecting a shape into multiple tables

With the `tableprojection` node defined in the `outputs` section above, you can slice parts of the `tableprojection` node into individual, related tables:

```json
"projections": [
    {
        "tables": [
            {
                "tableName": "hotelReviewsDocument",
                "generatedKeyName": "Documentid",
                "source": "/document/tableprojection"
            },
            {
                "tableName": "hotelReviewsPages",
                "generatedKeyName": "Pagesid",
                "source": "/document/tableprojection/pages/*"
            },
            {
                "tableName": "hotelReviewsKeyPhrases",
                "generatedKeyName": "KeyPhrasesid",
                "source": "/document/tableprojection/pages/*/keyphrase/*"
            }
        ]
    }
]
```

## Inline shaping projections

Inline shaping is the ability to form new shapes within the projection definition itself. Inline shaping has these characteristics:

+ The shape can be used only by the projection that contains it.
+ The shape can be identical to what a Shaper skill would produce.

An inline shape is created using `sourceContext` and `inputs`.

| Property | Description |
|----------|-------------|
| sourceContext | Sets the root of the projection.  |
| inputs | Each input is a column in the table. Name is the column name. Source is the enrichment node that provides the value. |

To project the same data as the previous example, the inline projection option would look like this:

```json
"projections": [
    {
        "tables": [
            {
                "tableName": "hotelReviewsInlineDocument",
                "generatedKeyName": "Documentid",
                "sourceContext": "/document",     
                "inputs": [
                    {
                        "name": "reviews_text",
                        "source": "/document/reviews_text"
                    },
                    {
                        "name": "reviews_title",
                        "source": "/document/reviews_title"
                    },
                    {
                        "name": "AzureSearch_DocumentKey",
                        "source": "/document/AzureSearch_DocumentKey"
                    }                             
                ]
            },
            {
                "tableName": "hotelReviewsInlinePages",
                "generatedKeyName": "Pagesid",
                "sourceContext": "/document/reviews_text/pages/*",
                "inputs": [
                        {
                    "name": "Sentiment",
                    "source": "/document/reviews_text/pages/*/Sentiment"
                    },
                    {
                        "name": "LanguageCode",
                        "source": "/document/Language"
                    },
                    {
                        "name": "Page",
                        "source": "/document/reviews_text/pages/*"
                    }
                ]
            },
            {
                "tableName": "hotelReviewsInlineKeyPhrases",
                "generatedKeyName": "KeyPhraseId",
                "sourceContext": "/document/reviews_text/pages/*/Keyphrases/*",
                "inputs": [
                    {
                        "name": "Keyphrases",
                        "source": "/document/reviews_text/pages/*/Keyphrases/*"
                    }
                ]
            }
        ]
    }
]
```
  
One observation from both the approaches is how values of "Keyphrases" are projected using the "sourceContext". The "Keyphrases" node, which contains a collection of strings, is itself a child of the page text. However, because projections require a JSON object and the page is a primitive (string), the "sourceContext" is used to wrap the key phrase into an object with a named property. This technique enables even primitives to be projected independently.

## Next steps

This article describes the concepts and principles of projection shapes. As a next step, see how these are applied in patterns for table, object, and file projections.

> [!div class="nextstepaction"]
> [Define projections in a knowledge store](knowledge-store-projections-examples.md)
