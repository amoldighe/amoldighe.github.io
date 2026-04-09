---
layout: post
title:  "Postgres Disk & WAL"
date:   2026-04-09
tags:
  - Postgres
  - Database
  - RDBMS
  - ACID
  - Transaction
  - Isolation
  - Durability
  - MVCC
  - WAL
---

Why is there size difference in my postgres cluster data directory?

I have a postgres patroni cluster with 3 nodes. Each node has a seperate data directory for postgres.

While verifiying the status of the cluster it was noticed that the cluster replica were stuck in a `starting` state.

```
[root@postgres-poc-node-3:~]# patronictl list
+ Cluster: postgres-poc-node (7587706677345667891) -------------------------+---------+----------+-----+-------------+-----+------------+-----+
| Member                             | Host                                                | Role    | State    |  TL | Receive LSN | Lag | Replay LSN | Lag |
+------------------------------------+-----------------------------------------------------+---------+----------+-----+-------------+-----+------------+-----+
| postgres-poc-node-1 | postgres-poc-node-1.localhost:5432 | Replica | starting |     |     unknown |     |    unknown |     |
| postgres-poc-node-2 | postgres-poc-node-2.localhost:5432 | Replica | starting |     |     unknown |     |    unknown |     |
| postgres-poc-node-3 | postgres-poc-node-3.localhost:5432 | Leader  | running  | 135 |             |     |            |     |
+------------------------------------+-----------------------------------------------------+---------+----------+-----+-------------+-----+------------+-----+
```

Each of the replica nodes had to be reinitialized to bring them back to a running state.


```
[root@postgres-poc-node-2:~]# patronictl reinit postgres-poc-node postgres-poc-node-2
+ Cluster: postgres-poc-node (7587706677345667891) -------------------------+---------+----------+-----+-------------+-----+------------+-----+
| Member                             | Host                                                | Role    | State    |  TL | Receive LSN | Lag | Replay LSN | Lag |
+------------------------------------+-----------------------------------------------------+---------+----------+-----+-------------+-----+------------+-----+
| postgres-poc-node-1 | postgres-poc-node-1.localhost:5432 | Replica | starting |     |     unknown |     |    unknown |     |
| postgres-poc-node-2 | postgres-poc-node-2.localhost:5432 | Replica | starting |     |     unknown |     |    unknown |     |
| postgres-poc-node-3 | postgres-poc-node-3.localhost:5432 | Leader  | running  | 135 |             |     |            |     |
+------------------------------------+-----------------------------------------------------+---------+----------+-----+-------------+-----+------------+-----+
Are you sure you want to reinitialize members postgres-poc-node-2? [y/N]: y
Success: reinitialize for member postgres-poc-node-2
```

Once all the replica nodes were back online, I noticed that the data directory size on the replica nodes was significantly smaller than the leader node.

* leader node

```
[root@postgres-poc-node-3:~]# du -sh /var/lib/postgresql/*/* | sort -h | tail -20
4.0K	/var/lib/postgresql/17/pg_stat_tmp
4.0K	/var/lib/postgresql/17/pg_tblspc
4.0K	/var/lib/postgresql/17/pg_twophase
4.0K	/var/lib/postgresql/17/PG_VERSION
4.0K	/var/lib/postgresql/17/postgresql.auto.conf
4.0K	/var/lib/postgresql/17/postgresql.conf
4.0K	/var/lib/postgresql/17/postgresql.conf.backup
4.0K	/var/lib/postgresql/17/postmaster.opts
4.0K	/var/lib/postgresql/17/postmaster.pid
4.0K	/var/lib/postgresql/patroni/pgpass
16K	/var/lib/postgresql/17/pg_logical
20K	/var/lib/postgresql/17/pg_replslot
32K	/var/lib/postgresql/17/postgresql.base.conf
32K	/var/lib/postgresql/17/postgresql.base.conf.backup
52K	/var/lib/postgresql/17/pg_multixact
600K	/var/lib/postgresql/17/global
15M	/var/lib/postgresql/17/pg_xact
31M	/var/lib/postgresql/17/pg_subtrans
3.3G	/var/lib/postgresql/17/base
16G	/var/lib/postgresql/17/pg_wal
```

* replica node

```
[root@postgres-poc-node-2:~]# du -sh /var/lib/postgresql/*/* | sort -h | tail -20
4.0K	/var/lib/postgresql/17/pg_tblspc
4.0K	/var/lib/postgresql/17/pg_twophase
4.0K	/var/lib/postgresql/17/PG_VERSION
4.0K	/var/lib/postgresql/17/postgresql.auto.conf
4.0K	/var/lib/postgresql/17/postgresql.conf
4.0K	/var/lib/postgresql/17/postgresql.conf.backup
4.0K	/var/lib/postgresql/17/postmaster.opts
4.0K	/var/lib/postgresql/17/postmaster.pid
4.0K	/var/lib/postgresql/patroni/pgpass
16K	/var/lib/postgresql/17/pg_logical
20K	/var/lib/postgresql/17/pg_replslot
28K	/var/lib/postgresql/17/pg_subtrans
32K	/var/lib/postgresql/17/postgresql.base.conf
32K	/var/lib/postgresql/17/postgresql.base.conf.backup
52K	/var/lib/postgresql/17/pg_multixact
232K	/var/lib/postgresql/17/backup_manifest
568K	/var/lib/postgresql/17/global
15M	/var/lib/postgresql/17/pg_xact
33M	/var/lib/postgresql/17/pg_wal
3.3G	/var/lib/postgresql/17/base
```

`base` contains the actual table and index files. Both nodes have about 3.3 GB there, so the actual database contents are roughly identical.

The large difference is entirely from `pg_wal`:

* Leader: 16 GB WAL retained
* Replica: 33 MB WAL retained

This is expected because replicas replay WAL and discard old WAL locally, while the leader must retain WAL for replicas, crash recovery, PITR, and replication slots.

Write-Ahead Logging is PostgreSQL’s durability and replication mechanism.

Basic idea:

* A client changes data:
`update orders set status='done' where id=10;`
* PostgreSQL first writes a WAL record describing the change into pg_wal
* Only after WAL is safely flushed does PostgreSQL consider the transaction committed
* Later, the actual table file in base/ is updated

This is called "write-ahead" because WAL is written before the actual data files.

Why this exists?

* crash recovery
* replication
* point-in-time recovery
* consistency during restart

In this case as the replication was broken for a long time hence the leader node had to retain all the WAL files for the replicas to catch up. 

Once the cluster was healthy again the replicas started catching up and the WAL files were discarded from the leader node. 

```
[root@postgres-poc-node-3:~]# patronictl list
+ Cluster: postgres-poc-node (7587706677345667891) -------------------------+---------+-----------+-----+-------------+-----+------------+-----+
| Member                             | Host                                                | Role    | State     |  TL | Receive LSN | Lag | Replay LSN | Lag |
+------------------------------------+-----------------------------------------------------+---------+-----------+-----+-------------+-----+------------+-----+
| postgres-poc-node-1 | postgres-poc-node-1.localhost:5432 | Replica | streaming | 135 |  D/7F9BD988 |   0 |    unknown |     |
| postgres-poc-node-2 | postgres-poc-node-2.localhost:5432 | Replica | streaming | 135 |  D/7F9BD988 |   0 | D/7F9BD988 |   0 |
| postgres-poc-node-3 | postgres-poc-node-3.localhost:5432 | Leader  | running   | 135 |             |     |            |     |
+------------------------------------+-----------------------------------------------------+---------+-----------+-----+-------------+-----+------------+-----+

[root@postgres-poc-node-3:~]# du -sh /var/lib/postgresql/*/* | sort -h | tail -20
4.0K	/var/lib/postgresql/17/pg_stat_tmp
4.0K	/var/lib/postgresql/17/pg_tblspc
4.0K	/var/lib/postgresql/17/pg_twophase
4.0K	/var/lib/postgresql/17/PG_VERSION
4.0K	/var/lib/postgresql/17/postgresql.auto.conf
4.0K	/var/lib/postgresql/17/postgresql.conf
4.0K	/var/lib/postgresql/17/postgresql.conf.backup
4.0K	/var/lib/postgresql/17/postmaster.opts
4.0K	/var/lib/postgresql/17/postmaster.pid
4.0K	/var/lib/postgresql/patroni/pgpass
16K	/var/lib/postgresql/17/pg_logical
20K	/var/lib/postgresql/17/pg_replslot
28K	/var/lib/postgresql/17/pg_subtrans
32K	/var/lib/postgresql/17/postgresql.base.conf
32K	/var/lib/postgresql/17/postgresql.base.conf.backup
52K	/var/lib/postgresql/17/pg_multixact
600K	/var/lib/postgresql/17/global
15M	/var/lib/postgresql/17/pg_xact
337M	/var/lib/postgresql/17/pg_wal
3.3G	/var/lib/postgresql/17/base
```
