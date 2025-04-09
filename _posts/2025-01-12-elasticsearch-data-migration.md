---
layout: post
title:  "Elasticsearch Data Migration"
date:   2025-01-12
tags:
  - elk
  - elasticsearch
  - devtools
  - kibana
  - migrate
  - index
  - elasticsearch_dump
  - remote reindexing
  - snapshot
  - backup-restore
  - curl
---


### Snapshot & Restore 

Snapshot feature in Elasticsearch allows taking backup of complete Elasticsearch cluster data.

Consecutive backups are incremental snapshots


* Define the snapshot repository path 

```
vi /etc/elasticsearch/elasticsearch.yml
   path.repo: /mnt/esbackup/es_backup
```

* Register a snapshot repository 

```
PUT /_snapshot/es_backup
{
  "type": "fs",
  "settings": {
    "location": "/mnt/esbackup/es_backup"
  }
}
```

* Create an snapshot manually 

```
PUT _snapshot/es_backup/my_snapshot
```

This will run a snapshot process in background 

* Status of a snapshot process 

```
GET _snapshot/my_repository/_current

GET _snapshot/_status
```

* List the snapshots available on the data path

```
GET _snapshot/es_backup/_all
```

This will list the snapshot name & the indices name

* Restore an indes from snapshot

```
POST /_snapshot/es_backup/snapshot20190502/_restore
{
  "indices": "<index name>>",
  "include_aliases": false,
  "ignore_unavailable": false,
  "include_global_state": false,
  "rename_pattern": "[a-zA-Z0-9-]+",
  "rename_replacement": "restored_index_$0"
}
```


### Reindex to a remote cluster

* Whitelist the remote host in elasticsearch.yml 

```
reindex.remote.whitelist: "otherhost:9200"
```

* Start the reindex process

```
POST _reindex
{
  "source": {
    "remote": {
      "host": "http://otherhost:9200",
      "username": "user",
      "password": "pass"
    },
    "index": "my-index-000001",
    "query": {
      "match": {
        "test": "data"
      }
    }
  },
  "dest": {
    "index": "my-new-index-000001"
  }
}
```


### Dump Elasticsearch index data as JSON file

* Download elasticsearch-dump docker container from [https://hub.docker.com/r/elasticdump/elasticsearch-dump/](https://hub.docker.com/r/elasticdump/elasticsearch-dump/)

```
~ » docker pull elasticdump/elasticsearch-dump                                                                         
Using default tag: latest
latest: Pulling from elasticdump/elasticsearch-dump
4d2547c08499: Pull complete
2476b939e842: Pull complete
5abb1ac0e1b3: Pull complete
91082d37f79b: Pull complete
348849d14ed6: Pull complete
18108483bdc3: Pull complete
6fdfb6e9a923: Pull complete
dc60419a52c6: Pull complete
Digest: sha256:4eb4eb891c54314cf4b97f698d4772abdc4269f427f839bfd5c0d6364b009fc4
Status: Downloaded newer image for elasticdump/elasticsearch-dump:latest
docker.io/elasticdump/elasticsearch-dump:latest
```

* Run the docker container to backup the required INDEX 

```
~ » docker run --rm -ti -v /Users/amol/ES_DATA:/tmp elasticdump/elasticsearch-dump \                                           
\ --input=http://<ES CLUSTER DNS>:9200/INDEX-2025.03.25 \
\ --output=/ES_DATA/INDEX-2025.03.25.json \
\ --type=data
Wed, 26 Mar 2025 12:21:16 GMT | starting dump
Wed, 26 Mar 2025 12:21:17 GMT | got 14 objects from source elasticsearch (offset: 0)
Wed, 26 Mar 2025 12:21:17 GMT | sent 14 objects, 0 offset, to destination file, wrote 14
Wed, 26 Mar 2025 12:21:17 GMT | got 0 objects from source elasticsearch (offset: 100)
Wed, 26 Mar 2025 12:21:17 GMT | Total Writes: 14
Wed, 26 Mar 2025 12:21:17 GMT | dump complete
```

* Once index backup is complete, the container shutsdown. 
Backup is available on the system mounted path


```
~/ES_DATA» ls -lth                                                                                                               
total 24
-rw-r--r--  1 amol  data   9.5K 26 Mar 17:51 INDEX-2025.03.25.json
```