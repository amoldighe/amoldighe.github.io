---
layout: post
title:  "Elasticsearch Cheatsheet using CURL"
date:   2024-7-21
tags:
  - elk
  - elasticsearch
  - devtools
  - kibana
  - deploy-keys
  - index
  - deploy
  -curl
---


### List open indexes
```
curl -X GET http:/ES_URL:9200/_cat/indices | grep open
```

### List close indexes
```
curl -X GET http:/ES_URL:9200/_cat/indices | grep close
```

### List close indexes, display only index name 
```
curl -X GET http:/ES_URL:9200/_cat/indices | grep close | sort | awk '{print $2}'
```

### Delete index starting with index pattern
```
curl -X DELETE http:/ES_URL:9200/index_pattern\*
```

### Close a list of indexes 
```
for i in `cat ES_INDEX_LIST_FILE`;do curl -X POST http:/ES_URL:9200/$i/_close; done
```

### Delete a list of indexes 
```
for i in `cat ES_INDEX_LIST_FILE`;do curl -X DELETE http:/ES_URL:9200/$i; done
```

### Insert data into Elasticsearch
```
curl -XPOST 'http://ES_URL:9200/test_index-test/_doc' \                            
\ -H 'Content-Type: application/json' \
\ -u user:password \
\ -d '{
    "field1": "value1",
    "field2": "value2"
  }'
```

### Stop / Start cluster rebalancing
```
curl -X PUT http://ES_URL:9200/_cluster/settings -H "Content-Type: application/json" -d'
    {
      "transient": {
        "cluster.routing.allocation.enable": "none"
      }
    }

curl -X GET http://ES IP:9200/_cluster/settings

curl -X POST http://ES IP:9200/_flush/synced


curl -X PUT http://ES IP:9200/_cluster/settings -H "Content-Type: application/json" -d'
    {
      "transient": {
        "cluster.routing.allocation.enable": "all"
      }
    }'
```

