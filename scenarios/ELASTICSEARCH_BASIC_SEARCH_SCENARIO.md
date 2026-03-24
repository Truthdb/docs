# Elasticsearch Basic Search Scenario

## Goal

Stand up Elasticsearch and prove the basic document-search workflow:

- create an index
- insert a small set of JSON documents
- run a text search
- run a filtered search

This is the smallest useful Elasticsearch scenario because it validates the core behavior Elasticsearch is known for: indexing documents and searching them by relevance.

## Scenario

Assume Elasticsearch has just been installed and is running on `localhost:9200`.

The simplest realistic dataset is a tiny product catalog. We will create one `products` index, insert three documents, then run a couple of searches against them.

## Step 1: Create The Index

```sh
curl -X PUT http://localhost:9200/products \
  -H 'Content-Type: application/json' \
  -d '{
    "mappings": {
      "properties": {
        "name": { "type": "text" },
        "category": { "type": "keyword" },
        "price": { "type": "float" },
        "description": { "type": "text" }
      }
    }
  }'
```

## Step 2: Insert A Few Documents

```sh
curl -X POST http://localhost:9200/products/_doc/1?refresh=true \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "Red Running Shoes",
    "category": "shoes",
    "price": 79.99,
    "description": "Lightweight shoes for road running"
  }'

curl -X POST http://localhost:9200/products/_doc/2?refresh=true \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "Blue Hiking Boots",
    "category": "boots",
    "price": 129.99,
    "description": "Durable boots for mountain trails"
  }'

curl -X POST http://localhost:9200/products/_doc/3?refresh=true \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "Black Sneakers",
    "category": "shoes",
    "price": 59.99,
    "description": "Comfortable everyday sneakers"
  }'
```

Using `refresh=true` keeps the scenario simple by making each document searchable immediately.

## Step 3: Run A Basic Text Search

```sh
curl -X GET http://localhost:9200/products/_search \
  -H 'Content-Type: application/json' \
  -d '{
    "query": {
      "match": {
        "description": "running shoes"
      }
    }
  }'
```

Expected outcome:

- the running shoes document should be returned
- results are ranked by relevance rather than returned in insertion order

## Step 4: Run A Filtered Search

```sh
curl -X GET http://localhost:9200/products/_search \
  -H 'Content-Type: application/json' \
  -d '{
    "query": {
      "bool": {
        "must": [
          { "match": { "description": "comfortable" } }
        ],
        "filter": [
          { "term": { "category": "shoes" } }
        ]
      }
    }
  }'
```

Expected outcome:

- only products in the `shoes` category are eligible
- matching text in `description` still affects relevance

## Success Criteria

This scenario is successful if:

- the index can be created
- documents can be inserted
- a `match` query returns relevant documents
- a filtered query narrows the result set correctly

## Why This Is The Standard First Scenario

This scenario avoids advanced Elasticsearch features such as clustering, shard tuning, aggregations, pipelines, or lifecycle policies. It focuses on the first thing most users want to verify after installation: "Can I put documents in and search them in a useful way?"
