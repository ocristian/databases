# PostgreSQL Utils

## Database

### Activity
```sql
  SELECT distinct datname, count(*)
    FROM pg_stat_activity
   WHERE datname = 'database_name'
GROUP BY datname
ORDER BY 1;
```
### Killing Active connections
```sql
SELECT pg_terminate_backend(pg_stat_activity.pid)
  FROM pg_stat_activity
 WHERE pg_stat_activity.datname = 'database_name'
   AND pid <> pg_backend_pid();
```

### TOP 10
#### Time Consuming
```sql
  SELECT userid::regrole as "user",
         datname as database,
         calls,
         round(mean_time::numeric, 2) as "Avg Time (ms)",
         round(total_time::numeric, 2) as "Total Time (ms)",
         query
    FROM pg_stat_statements join pg_database
            on pg_stat_statements.dbid = pg_database.oid
ORDER BY mean_time DESC
   LIMIT 10;
```
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


### Listing databases
```sql
SELECT * FROM pg_database pd WHERE datallowconn;
SELECT datname FROM pg_database pd WHERE datallowconn AND datname NOT IN ('rdsadmin', 'template1');
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
```

## Extensions

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
  SELECT n.nspname AS "Name", pg_catalog.pg_get_userbyid(n.nspowner) AS "Owner"
    FROM pg_catalog.pg_namespace n
   WHERE n.nspname !~ '^pg_' AND n.nspname <> 'information_schema'
ORDER BY 1;
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
### Updating
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
