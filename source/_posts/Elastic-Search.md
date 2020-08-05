---
title: Elastic Search
date: 2019-07-28 01:20:37
tags:
- Web
- Java
categories: es
hidden: true
---


# 1. concept

## Document
A document is a JSON document which is stored in Elasticsearch. It is like a row in a table in a relational database. Each document is stored in an index and has a type and an id.

JSON object is composed by Field 由字段组成
- String: text, keyword
- Number: long, integer, short, byte, double, float, half_float, scaled_float
- Boolean
- Date
- Binary
- Range: integer_range, float_range...

Each document has an unique id (like primary key in db).

A sample: field name and field value
```json
{
    "remote_ip": "xxx.xx.xxx.xxx",
    "user_name": "-",
    "request_action": "GET",
    "request": "/downloads/product_1",
    "http_version":"1.1"
    //...
}
```

Document also has metadata.
- _index: the index name 
- _type: type name
- _id
- _uid 
- _source
- _all

## Index
An index is like a table in a relational database. Index has schema. It's called mapping which defines field name and field type. A cluster can have multiple indexes.
- nginx can generate a separate index per day.

## Node
A running instance

## Cluster
multiple instances

## REST
Index and Document are resources.

- Index
  - PUT /test_index // create index

  - GET _cat/indices

  - DELETE /test_index

  - PUT /test_index

- Document
  - PUT /test_index/doc/1    // doc here is type. 1 is id. If index doesn't exist. It'll automatically create one.
  
  ```json
   {
        "username" : "alfred",
        "age" : 1        
   }
   ```

   - POST /test_index/doc   // without id. Use post
   - GET /test_index/doc/1
   - GET /test_index/doc/_search    // search all documents
     ```json
        {
            "query": {
                "term" : {
                    "_id": "1"
                }
            }
        }
     ```

    - POST _bulk 


## Inverted Index
Inverted index is composed by
- Term Dictionary 单词词典
  - it records all words
  - it records association relationship between word and posting list 
- Posting List 倒排列表

## Term or token

