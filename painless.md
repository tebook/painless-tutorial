# painless-tutorial

Most of the time if you are using Elasticsearch(ES), data may have to transformed before ingesting. A dynamic programming language "Painless" was created exclusively for ES which can help with cleaning, enriching and tranforming data.

If you are preparing for the Elastic Certified Engineer's Exam, working knowledge of Painless becomes extremely important. The following section discuss about the basics of Painless to get you started quickly. For this tutorial I have used the accounts index which can be created using the sample data found here.

https://github.com/tebook/prepbook-elastic-certified-engineer/blob/main/elastprepbook/accounts.json

Use Data Visualizer under Machine Learning to ingest the accounts.json file into ES. I am also using the set processor to set doc _id same as the account number field for each document. This can be done using ingest pipeline setting which can be found under Advanced section.

![image](https://user-images.githubusercontent.com/99671188/162035534-a3b3a15d-3c56-4122-b7df-c62bc7954b35.png)


Let's have a look at the syntax of painless scripts before we start using them.
```
"script": {
 "lang": "...", 
 "source": "...",
 "params": { ... }
 }
```
The script syntax is divided into three components, as seen above:

* lang: The language in which the script is written. Because the default is painless, this parameter is not required if painless scripts are used. "painless," "expression," "moustache," "java," are examples of permissible values for this field.
* source: The real script will be found in the source section.
* params: If any params are required by the defined script, they can be given in this section.

Depending on where a painless script is used, it will have access to certain special variables and document fields. A script used in the update, update-by-query, or reindex API will have access to the ctx variable which exposes document fields as ctx._source.fieldname.

Scripts used in search and aggregations will be executed once for every document that matches a query or an aggregation, with the exception of script fields(discussed later in this tutorial), which are executed once per search hit. This might be millions or billions of executions, depending on how many documents you have: these scripts must be fast! Here doc-values, the _source field, and saved fields can all be used to get field values from a script.

Painless can be used with the _update API to update the documents of an index. As mentioned above with update, update-by-query, or reindex API the way of accessing a value is by using the syntax ctx._source.field name (i.e. use "_source" fields). For this example I am taking a document with doc _id 6 and changing the gender field to Male.

```
POST accounts/_update/6
{
  "script": {
    "source": "ctx._source.gender = 'Male'"
  }
}
```
Let's verify that the document has been updated by making use of the _doc API

```
GET accounts/_doc/6
```

Because the default language in ES is "painless," there was no need to specify the "lang" option in the previous example. We didn't send any parameters to the scripts, therefore "params" was left out as well. What would be the benefit of using "params"? Other than hardcoding the data in the script, why should you use "params"?


Elasticsearch builds a new script the first time it sees it and saves the built version in a cache. Compilation is a time-consuming procedure. Every time the value of the gender field is changed (for example, in the preceding example, when we change the value of the gender field to Male), the script must be recompiled. However, if we expect the script to use dynamic parameters (variables), we should send them using the "params" field. The "params" script would only have to be compiled once.

Let's look at how to change the example above to pass variables to the script:
```
POST accounts/_update/1
{
  "script": {
    "source": "ctx._source.gender = params.F",
    "params": {
      "F" : "Female"
    }
  }
}
```

```
GET accounts/_doc/1
```


Painless can also be used with _update_by_query. In the last example we changed gender for few documents, lets change gender from M to Male for all the male accounts. The _update_by_query as the name implies will run an update on documents based on the outcome of the query.
```
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
```
Another common use of Painless is in ingest pipelines. Lets create a ingest pipeline by the name demo and use the script processor to change the gender back to M and F from Male and Female.
```
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
```
You must have noticed that when used with script processor the feild values must be accessed using ctx.fieldname and not ctx._source.fieldname. It can throw you off at times, so do remember it for the exam.

Let us call this pipeline using the _reindex API and reindex to a new index calling it the bank index.
```
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
```
Painless can also be used for search and aggregation operations. As previously mentioned doc values could be used for search and aggregation operations. 

Lets perform a simple search operation and sort the results by firstname in descending order. 
```
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
```
A common use case is to have to build dynamic fields on the fly while searching. You can build dynamic fields in the query answer using script fields ("script fields"). Let's make a scripted field called balancebyage that is formed by dividing account balance by the account holder's age, and observe how script fields work. Also, because the balance and age fields are not analyzed and have doc values enabled by default, I'm getting them via doc values rather than _source, which would be substantially slower.
```
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
```


Hope you find this tutorial useful. I will updating it by adding more examples very soon.

If you are preparing for the Elastic Certified Engineer Exam, do checkout this book written by me

https://www.amazon.com/Exam-Guide-Elastic-Certified-Engineer-ebook/dp/B09TQ1PS6T/
