---
layout: post
title:  "Elasticsearch Cheatsheet"
date:   2024-7-21
tags:
  - elk
  - elasticsearch
  - devtools
  - kibana
  - deploy-keys
  - index
  - deploy
---

### Create New Index 

```
PUT index_name
```

### Create Index Teamplate 

```
PUT _index_template/index_name
{
  "index_patterns": ["index_name*"], 
  "allow_auto_create": true,
  "priority": 100,
  "template": {
    "settings": {
      "number_of_shards": 4,
      "number_of_replicas": 1,
      "index.lifecycle.name": "testindex-ilm",
      "index.lifecycle.rollover_alias": "index_name_main"
    }
  }
}
```

### Create bootstrap index

```
PUT amolindex_rpm_url_index_main-000001
{
  "aliases": {
    "amolindex_rpm_url_index_main": {
      "is_write_index": true
    }
  }
}
```

### Delete index 
```
DELETE dm-logstash-apache-access-log
```
### List all indexs
```
GET /_cat/indices/*
```
### List all indexs with shards 
```
GET _cat/shards?v=true
```
###  List all shards for specific index
```
GET _cat/shards?v=true&index=test_index*
```
### Display elasticsearch node details 
```
GET _cat/nodes?v=true&
```
### Display elasticsearch nodes heap.max
```
GET _cat/nodes?v=true&h=heap.max
```
### Display elasticsearch node JVM details 
```
GET _nodes/stats?human&filter_path=nodes.*.name,nodes.*.indices.mappings.total_estimated_overhead*,nodes.*.jvm.mem.heap_max*
```
### Diplay index details
```
GET /_cat/indices/test-index-name*?v
```
### Display index stats
```
GET new-index/_stats
```
### Diplay index details with parameter specification
```
GET /_cat/indices/*test-logstash-apache-access*?h=index,pri,rep,store.size&bytes=gb
```
### Display index metadata
```
GET /new-index/_settings
```
### Display cluster health
```
GET /_cat/health
```
### Display cluster settings
```
GET _cluster/settings
```
### Change index to readonly 
```
PUT amolindex_typ_tag_vw_index/_settings
{
  "index": {
    "blocks.read_only": true
  }
}
```
### Attach ILM to an index
```
POST test_index-metrics-prog-2024.10.*/_ilm/remove
```
### Remove ILM from an index 
```
PUT test_index-metrics-prog-2024.10.*/_settings
{
  "index": {
    "lifecycle": {
      "name": "test_index-metric-policy"
    }
  }
}
```

### Stop Cluster rebalancing 
```
PUT _cluster/settings
{
  "transient": {
    "cluster.routing.rebalance.enable": "none"
  }
}
```
### Start Cluster rebalancing 
```
PUT _cluster/settings
{
  "transient": {
    "cluster.routing.rebalance.enable": "all"
  }
}
```





