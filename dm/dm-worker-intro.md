---
title: DM-worker Introduction
summary: Learn the features of DM-worker.
aliases: ['/docs/tidb-data-migration/dev/dm-worker-intro/']
---

# DM-worker Introduction

DM-worker is a tool used to migrate data from MySQL/MariaDB to TiDB.

It has the following features:

- Acts as a secondary database of any MySQL or MariaDB instance
- Reads the binlog events from MySQL/MariaDB and persists them to the local storage
- A single DM-worker supports migrating the data of one MySQL/MariaDB instance to multiple TiDB instances
- Multiple DM-workers support migrating the data of multiple MySQL/MariaDB instances to one TiDB instance

## DM-worker processing unit

A DM-worker task contains multiple logic units, including relay log, the dump processing unit, the load processing unit, and binlog replication.

### Relay log

The relay log persistently stores the binlog data from the upstream MySQL/MariaDB and provides the feature of accessing binlog events for the binlog replication.

Its rationale and features are similar to the relay log of MySQL. For details, see [MySQL Relay Log](https://dev.mysql.com/doc/refman/5.7/en/replica-logs-relaylog.html).

### Dump processing unit

The dump processing unit dumps the full data from the upstream MySQL/MariaDB to the local disk.

### Load processing unit

The load processing unit reads the dumped files of the dump processing unit and then loads these files to the downstream TiDB.

### Binlog replication/sync processing unit

Binlog replication/sync processing unit reads the binlog events of the upstream MySQL/MariaDB or the binlog events of the relay log, transforms these events to SQL statements, and then applies these statements to the downstream TiDB.

## Privileges required by DM-worker

This section describes the upstream and downstream database users' privileges required by DM-worker, and the user privileges required by the respective processing unit.

### Upstream database user privileges

The upstream database (MySQL/MariaDB) user must have the following privileges:

| Privilege | Scope |
|:----|:----|
| `SELECT` | Tables |
| `RELOAD` | Global |
| `REPLICATION SLAVE` | Global |
| `REPLICATION CLIENT` | Global |

If you need to migrate the data from `db1` to TiDB, execute the following `GRANT` statement:

```sql
GRANT RELOAD,REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'your_user'@'your_wildcard_of_host'
GRANT SELECT ON db1.* TO 'your_user'@'your_wildcard_of_host';
```

If you also need to migrate the data from other databases into TiDB, make sure the same privileges are granted to the user of the respective databases.

### Downstream database user privileges

The downstream database (TiDB) user must have the following privileges:

| Privilege | Scope |
|:----|:----|
| `SELECT` | Tables |
| `INSERT` | Tables |
| `UPDATE` | Tables |
| `DELETE` | Tables |
| `CREATE` | Databases, tables |
| `DROP` | Databases, tables |
| `ALTER` | Tables |
| `INDEX` | Tables |

Execute the following `GRANT` statement for the databases or tables that you need to migrate:

```sql
GRANT SELECT,INSERT,UPDATE,DELETE,CREATE,DROP,ALTER,INDEX ON db.table TO 'your_user'@'your_wildcard_of_host';
GRANT ALL ON dm_meta.* TO 'your_user'@'your_wildcard_of_host';
```

### Minimal privilege required by each processing unit

| Processing unit | Minimal upstream (MySQL/MariaDB) privilege | Minimal downstream (TiDB) privilege | Minimal system privilege |
|:----|:--------------------|:------------|:----|
| Relay log | `REPLICATION SLAVE` (reads the binlog)<br/>`REPLICATION CLIENT` (`show master status`, `show slave status`) | NULL | Read/Write local files |
| Dump | `SELECT`<br/>`RELOAD` (flushes tables with Read lock and unlocks tables）| NULL | Write local files |
| Load | NULL | `SELECT` (Query the checkpoint history)<br/>`CREATE` (creates a database/table)<br/>`DELETE` (deletes checkpoint)<br/>`INSERT` (Inserts the Dump data) | Read/Write local files |
| Binlog replication | `REPLICATION SLAVE` (reads the binlog)<br/>`REPLICATION CLIENT` (`show master status`, `show slave status`) | `SELECT` (shows the index and column)<br/>`INSERT` (DML)<br/>`UPDATE` (DML)<br/>`DELETE` (DML)<br/>`CREATE` (creates a database/table)<br/>`DROP` (drops databases/tables)<br/>`ALTER` (alters a table)<br/>`INDEX` (creates/drops an index)| Read/Write local files |

> **Note:**
>
> These privileges are not immutable and they change as the request changes.
