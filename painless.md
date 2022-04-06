# painless-tutorial

Most of the time if you are using Elasticsearch(ES), data may have to transformed before ingesting. A dynamic programming language "Painless" was created exclusively for ES which can help with cleaning, enriching and tranforming data.

If you are preparing for the Elasticsearch Certified Engineer's Exam, working knowledge of Painless becomes extremely important. The following section discuss about the basics of Painless to get you started quickly. For this tutorial I have used the accounts index which can be created using the sample data found here.

Let's have a look at the syntax of painless scripts before we start using them.

"script": {
 "lang": "...", 
 "source": "...",
 "params": { ... }
 }

The script syntax is divided into three components, as seen above:

lang: The language in which the script is written. Because the default is painless, this parameter is not required if painless scripts are used. "painless," "expression," "moustache," "java," are examples of permissible values for this field.
source: The real script will be found in the source section.
params: If any params are required by the defined script, they can be given in this section.

If doc values are enabled, the document values can be accessed within the script using the syntax doc['field name']. Doc values will be enabled by default for 'Not Analyzed' fields. Another way of accessing the value is by using the syntax ctx. source.field name (i.e. use "_source" fields).  Typically you would like to use doc values for search and aggregation operations, and "_source" fields for updates.

Painless can be used with the _update API to update the documents of an index. Make sure to change the doc _id in the example below with your document id.



POST accounts/_update/cBoP-H8Bi5LMwQDv6Xb6
{
  "script": {
    "source": "ctx._source.gender = \"Male\""
  }
}

GET accounts/_doc/cBoP-H8Bi5LMwQDv6Xb6


Because the default language in ES is "painless," there was no need to specify the "lang" option in the previous example. We didn't send any parameters to the scripts, therefore "params" was left out as well. What would be the benefit of using "params"? Other than hardcoding the data in the script, why should you use "params"?


Elasticsearch builds a new script the first time it sees it and saves the built version in a cache. Compilation is a time-consuming procedure. Every time the value of the gender field is changed (for example, in the preceding example, when we change the value of the gender field to Male), the script must be recompiled. However, if we expect the script to use dynamic parameters (variables), we should send them using the "params" field. The "params" script would only have to be compiled once.

Let's look at how to change the example above to pass variables to the script:

POST accounts/_update/dRoP-H8Bi5LMwQDv6Xb6
{
  "script": {
    "source": "ctx._source.gender = params.F",
    "params": {
      "F" : "Female"
    }
  }
}

GET accounts/_doc/dRoP-H8Bi5LMwQDv6Xb6

Again make sure to replace the doc _id above with your doc _id.

Painless can also be used with _update_by_query. In the last example we changed gender for few documents, lets change gender from M to Male for all the male accounts. The _update_by_query as the name implies will run an update on documents based on the outcome of the query.

POST accounts/_update_by_query
{
  "query": {
    "match": {
      "gender": "M"
    }
  },
  "script": {
    "source": "ctx._source.gender = params.M",
    "lang": "painless",
    "params": {
      "M": "Male"
    }
  }
}

Another common use of Painless is in ingest pipelines. Lets create a ingest pipeline by the name demo and use the script processor to change the gender back to M and F from Male and Female.

PUT _ingest/pipeline/demo
{
  "description": "Quick demo of painless",
  "processors": [
    {
      "script": {
        "lang": "painless",
        "source": """
        if (ctx.gender == 'Male')
         {ctx.gender = params.male}
        else
         {ctx.gender = params.female}
        """,
        "params": {
          "male": "M",
          "female": "F"
        }
      }
    }
  ]
}

You must have noticed that when used with pipelines the feild values must be accessed using ctx.fieldname and not ctx._source.fieldname. It can throw you off at times, so do remember it for the exam.

Let us call this pipeline using the _reindex API and reindex to a new index calling it the bank index.

POST _reindex
{
  "source": {
    "index": "accounts"
  },
  "dest": {
    "index": "bank",
    "pipeline": "demo"
  }
}

Painless can also be used for search and aggregation operations. As previously mentioned doc values could be used for search and aggregation operations. 

Lets perform a simple search operation and sort the results by firstname in descending order. 

GET accounts/_search
{
  "query": {
    "match_all": {}
  },
  "sort": {
    "_script": {
      "type": "string",
      "order": "desc",
      "script": {
        "lang": "painless",
        "source": "doc['firstname'].value"
      }
    }
  }
}

A common use case is to have to build dynamic fields on the fly while searching. You can build dynamic fields in the query answer using script fields ("script fields"). Let's make a scripted field called balancebyage that is formed by dividing account balance by the account holder's age, and observe how script fields work. Also, because the balance and age fields are not analyzed and have doc values enabled by default, I'm getting them via doc values rather than source, which would be substantially slower.

GET bank/_search
{
  "query": {
    "match_all": {}
  },
  "script_fields": {
    "balancebyage": {
      "script": {
        "lang": "painless",
        "source": "doc['balance'].value/doc['age'].value"
      }
    }
  }