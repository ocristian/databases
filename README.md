# PostgreSQL Utils

<!-- TOC depthFrom:2 -->
- [Database](#database)
  * [Activity](#activity)
  * [Killing Active connections](#killing-active-connections)
- [TOP 10](#top-10)
  * [Time Consuming](#time-consuming)
  * [MEM Consumption](#mem-consumption)
- [Listing databases](#listing-databases)
  * [Active databases per Owner](#active-databases-per-owner)
- [Indexes](#indexes)
- [Extensions](#extensions)
  * [Create](#extensions-create)
  * [Update](#extensions-update)
  * [Delete](#extensions-delete)
  * [List](#extensions-list)

<!-- /TOC -->

<a name="database"></a>
## Database

<a name="activity"></a>
### Activity
```sql
  SELECT distinct datname, count(*)
    FROM pg_stat_activity
   WHERE datname = 'database_name'
GROUP BY datname
ORDER BY 1;
```
<a name="killing-active-connections"></a>
### Killing Active connections
```sql
SELECT pg_terminate_backend(pg_stat_activity.pid)
  FROM pg_stat_activity
 WHERE pg_stat_activity.datname = 'database_name'
   AND pid <> pg_backend_pid();
```

<a name="top-10"></a>
### TOP 10
<a name="#time-consuming"></a>
#### Time Consuming
```sql
  SELECT userid::regrole as "user",
         datname as database,
         calls,
         round(mean_exec_time::numeric, 2) as "Avg Time (ms)",
         round(total_exec_time::numeric, 2) as "Total Time (ms)",
         query
    FROM pg_stat_statements join pg_database
            on pg_stat_statements.dbid = pg_database.oid
ORDER BY mean_exec_time DESC
   LIMIT 10;
```
> For PG v12 and below, remove `exec` from `mean_exec_time` and `total_exec_time`

<a name="#mem-consumption"></a>
#### MEM Consumption
```sql
  SELECT userid::regrole as "user",
         datname as database,
         calls,
         pg_size_pretty((shared_blks_hit + shared_blks_dirtied)) as mem,
         queryid,
         query
    FROM pg_stat_statements join pg_database
           on pg_stat_statements.dbid = pg_database.oid
ORDER BY (shared_blks_hit + shared_blks_dirtied) DESC
   LIMIT 10;
```

<a name="#listing-databases"></a>
### Listing databases
```sql
SELECT * FROM pg_database pd WHERE datallowconn;
SELECT datname FROM pg_database pd WHERE datallowconn AND datname NOT IN ('rdsadmin', 'template1');
```
<a name="#active-databases-per-owner"></a>
#### Active databases per Owner
```sql
  SELECT d.datname AS database, u.usename AS owner
    FROM pg_database d JOIN pg_user u ON d.datdba = u.usesysid
   WHERE datname IN (SELECT datname FROM pg_stat_activity WHERE state is not null)
ORDER BY datname;
```

### Version
```sql
SELECT version();
```
### User
```sql
SELECT current_user;
```
### Upgrading
[Upgrading the PostgreSQL DB engine for Amazon RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_UpgradeDBInstance.PostgreSQL.html#USER_UpgradeDBInstance.PostgreSQL.MajorVersion.Process)  
[Troubleshooting PostGIS Extensions issues when upgrading Postgres](https://repost.aws/knowledge-center/rds-postgresql-upgrade-postgis)


<a name="indexes"></a>
## Indexes

### List indexes
```sql
SELECT * FROM pg_indexes WHERE schemaname ='public'
```

### To find non-used indexes
* Consider deleting the Index if `index_scans_count` is very low
```sql
   SELECT idxstats.schemaname as schema_name,
          idxstats.relname AS table_name,
          indexrelname AS index_name,
          idxstats.idx_scan AS index_scans_count,
          pg_size_pretty(pg_relation_size(idxstats.indexrelid)) AS index_size
     FROM pg_stat_all_indexes AS idxstats
            JOIN pg_index i ON idxstats.indexrelid = i.indexrelid
    WHERE idxstats.schemaname = 'public'
  AND NOT i.indisunique   -- is not a UNIQUE index
 ORDER BY idxstats.idx_scan;
```

### Sequential scans
* `seq_scan_count`: A high number of sequential scans may indicate that appropriate indexes are missing or underutilized.
```sql
  SELECT tabstats.schemaname AS schema_name,
         tabstats.relname AS table_name,
         tabstats.seq_scan AS seq_scan_count,
         tabstats.idx_scan AS index_scan_count,
         pg_size_pretty(pg_total_relation_size(tabstats.relid)) AS table_size
    FROM pg_stat_all_tables AS tabstats
   WHERE tabstats.schemaname = 'public'
ORDER BY seq_scan_count DESC;
```

### Create Index / ReIndex Progress Report
[To follow the progress of Index/ReIndex](index/track-index-reindex-progress.md)



<a name="extensions"></a>
## Extensions

<a name="extensions-create"></a>
### Create
```sql
CREATE EXTENSION extension_name;
```
#### In a Schema
```sql
CREATE EXTENSION IF NOT EXISTS extension_name WITH SCHEMA schema_name;
```

#### Specific version
```sql
CREATE EXTENSION IF NOT EXISTS extension_name VERSION '0.0.1';
```

#### In a Schema and Specific version
```sql
CREATE EXTENSION IF NOT EXISTS extension_name WITH SCHEMA schema_name VERSION '0.0.1';
```

<a name="extensions-update"></a>
### Update
#### Upgrading Path
```sql
SELECT * FROM pg_extension_update_paths('extension_name') WHERE source='current_version_number' AND target NOT LIKE '%next%' AND source<target AND path LIKE '%--%';
```
#### To Specific version
```sql
ALTER EXTENSION extension_name UPDATE TO 'version_number';
```
#### To Last available version
```sql
ALTER EXTENSION extension_name UPDATE;
```
<a name="extensions-delete"></a>
### Delete
```sql
DROP EXTENSION extension_name;
```
<a name="extensions-list"></a>
### List Extensions
### Installed
```sql
SELECT * FROM pg_available_extensions WHERE installed_version IS NOT NULL order by 1;
```
### Installed outdated
```sql
SELECT * FROM pg_available_extensions WHERE default_version > installed_version;
```
### Available versions
```sql
SELECT * FROM pg_available_extension_versions WHERE name LIKE '%pg_stat_statements%';
```
### By Owner
```sql
SELECT e.extname AS "Extensions", n.nspname AS schema_name, u.usename AS "Owner"
  FROM pg_extension e
    JOIN pg_namespace n ON e.extnamespace = n.oid
    JOIN pg_user u ON e.extowner = u.usesysid;
```
### By Schema
```sql
 SELECT e.extname AS "Name",
        e.extversion AS "Version",
        n.nspname AS "Schema",
        c.description AS "Description"
   FROM pg_catalog.pg_extension e
          LEFT JOIN pg_catalog.pg_namespace n ON n.oid = e.extnamespace
          LEFT JOIN pg_catalog.pg_description c ON c.objoid = e.oid
          AND c.classoid = 'pg_catalog.pg_extension'::pg_catalog.regclass
   WHERE e.extname LIKE '%postgis%'
ORDER BY 1;
```


#### PostGIS 
* [Support Matrix](https://trac.osgeo.org/postgis/wiki/UsersWikiPostgreSQLPostGIS#PostGISSupportMatrix)
* [Obsolete Versions](https://trac.osgeo.org/postgis/wiki/PostGISObsoleteVersionsMatrix)

#### postgis_raster  
- Check: https://repost.aws/knowledge-center/rds-postgresql-upgrade-postgis
```sql
-- Installation Path
SELECT probin FROM pg_proc WHERE proname = 'postgis_raster_lib_version';

-- If installation Path is lower than new version, run:
SELECT postGIS_extensions_upgrade();
```
