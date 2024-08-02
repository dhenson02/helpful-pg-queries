# helpful-pg-queries
Helpful PostgreSQL queries

Stole a good chunk of these directly from [here (gist)](https://gist.github.com/rgreenjr/3637525)

# TOC
- [Show running queries](#show-running-queries)
- [Kill running query](#kill-running-query)
- [Kill idle query](#kill-idle-query)
- [Vacuum commands](#vacuum-commands)
- [Detailed info on running / idle queries](#detailed-info-on-running--idle-queries)
- [All databases and their sizes](#all-databases-and-their-sizes)
- [All tables and their sizes, ordered by schema then size](#all-tables-and-their-sizes-ordered-by-schema-then-size)
- [Better size query - tables, indexes, views, materialized views](#better-size-query---tables-indexes-views-materialized-views)
- [Cache hit rates](#cache-hit-rates)
- [Table index usage rates](#table-index-usage-rates)
- [Cached indexes](#cached-indexes)
- [DB temporary files, size and count](#db-temporary-files-size-and-count)
- [DROP ALL INDEXES ON A TABLE](#drop-all-indexes-on-a-table) 
- [DROP and then RECREATE indexes + DISABLE the TRIGGERs](#drop-and-then-recreate-indexes--disable-the-triggers)
- [Find and reset (to max value) ID sequences on all tables](#find-and-reset-to-max-value-id-sequences-on-all-tables)
- [Find the sequences not attached to ID columns](#find-the-sequences-not-attached-to-id-columns)
- [Dump database on remote host to file](#dump-database-on-remote-host-to-file)
- [Import dump into existing database](#import-dump-into-existing-database)
- [Get locked queries](#get-locked-queries)
- [Get long-running queries](#get-long-running-queries)
- [Get config file changes vs defaults](#get-config-file-changes-vs-defaults)
- [First, last, and first/last aggs](#first-last-and-firstlast-aggs)

## Show running queries
```sql
-- version >= 9.2
SELECT pid, age(clock_timestamp(), query_start), usename, query 
FROM pg_stat_activity 
WHERE query != '<IDLE>' AND query NOT ILIKE '%pg_stat_activity%' 
ORDER BY query_start desc;
```

## Kill running query
```sql
SELECT pg_cancel_backend(procpid);
```

## Kill idle query
```sql
SELECT pg_terminate_backend(procpid);
```

## Vacuum command
```sql
VACUUM (VERBOSE, ANALYZE);
VACUUM (FULL, VERBOSE, ANALYZE);
VACUUM (FULL, VERBOSE, ANALYZE) TABLE tablename;
```

## Detailed info on running / idle queries
```sql
SELECT * FROM pg_stat_activity WHERE query NOT LIKE '<%';
```

## All databases and their sizes
```sql
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname))
FROM
    pg_database
ORDER BY
    pg_database_size(datname) DESC;
```

## All tables and their sizes, ordered by schema then size
```sql
SELECT
    schemaname || '.' || tablename AS "tableName",
    pg_size_pretty(pg_table_size(schemaname || '.' || tablename)) AS "tableSize"
FROM
    pg_tables
ORDER BY
    schemaname ASC,
    pg_table_size(schemaname || '.' || tablename) DESC;
```

## Better size query - tables, indexes, views, materialized views
```sql
SELECT relname   AS objectname
     , relkind   AS objecttype
     , reltuples AS entries
     , pg_size_pretty(pg_table_size(oid)) AS size  -- depending - see below
FROM   pg_class
WHERE  relkind IN ('r', 'i', 'm', 'v')
ORDER  BY pg_table_size(oid) DESC;
```

## Cache hit rates
*should not be less than 0.99*
```sql
SELECT 
    pg_size_pretty(sum(heap_blks_read)) as heap_read, 
    pg_size_pretty(sum(heap_blks_hit))  as heap_hit, 
    pg_size_pretty((sum(heap_blks_hit) - sum(heap_blks_read)) / sum(heap_blks_hit)) as ratio
FROM pg_statio_user_tables;
```

## Table index usage rates
*should not be less than 0.99*
```sql
SELECT 
    relname, 
    100 * idx_scan / (seq_scan + idx_scan) percent_of_times_index_used,
    n_live_tup rows_in_table
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC,  percent_of_times_index_used DESC;
```

## Cached indexes
```sql
SELECT 
   pg_size_pretty(sum(idx_blks_read)) as idx_read, 
   pg_size_pretty(sum(idx_blks_hit))  as idx_hit, 
   pg_size_pretty((sum(idx_blks_hit) - sum(idx_blks_read)) / sum(idx_blks_hit)) as ratio
FROM pg_statio_user_indexes;
```

## DB temporary files, size and count
```sql
SELECT
    db.temp_files                       AS "Temporary files"
    , pg_size_pretty(db.temp_bytes)     AS "Size of temporary files"
    , db.datname
FROM
    pg_stat_database db
ORDER BY
    db.temp_bytes DESC;
```

## DROP ALL INDEXES ON A TABLE 
*(courtesy of Erwin Brandstetter - [https://stackoverflow.com/a/34011744](https://stackoverflow.com/a/34011744))*
```sql
-- also handle if no indexes exist
DO
$$
    DECLARE

        cmd TEXT;

    BEGIN
            SELECT
                'DROP INDEX ' || string_agg(indexrelid::REGCLASS::TEXT, ', ')
            INTO cmd
            FROM
                pg_index i
                LEFT JOIN pg_depend d ON d.objid = i.indexrelid
                    AND d.deptype = 'i'
            WHERE
                i.indrelid = 'patient_documents'::REGCLASS -- possibly schema-qualified
              AND d.objid IS NULL; -- no internal dependency
        
        IF ( cmd IS NOT NULL ) THEN
            EXECUTE cmd;
        END IF;
    END;
$$;
```

## DROP and then RECREATE indexes + DISABLE the TRIGGERs
```sql
DO $$
DECLARE

    drop_cmd TEXT;
    create_cmd TEXT;

BEGIN

    SELECT
        'DROP INDEX ' || STRING_AGG(indexrelid::REGCLASS::TEXT, ', ') || ';',
        STRING_AGG(pg_get_indexdef(indexrelid::regclass), '; ') || ';'
    INTO
        drop_cmd,
        create_cmd
    FROM
        pg_index i
    LEFT JOIN pg_depend d
        ON d.objid = i.indexrelid AND d.deptype = 'i'
    WHERE
        i.indrelid = 'studies'::REGCLASS
        AND d.objid IS NULL;

    IF ( drop_cmd IS NOT NULL ) THEN
        EXECUTE drop_cmd;
    END IF;

    ALTER TABLE studies 
        DISABLE TRIGGER ALL;

    -- ... do expensive stuff

    IF ( create_cmd IS NOT NULL ) THEN
        EXECUTE create_cmd;
    END IF;

    ALTER TABLE studies 
        ENABLE TRIGGER ALL;

END;
$$;
```

## Find and reset (to max value) ID sequences on all tables
*where applicable (that match [ROLE] and have been used at least once)*
```sql
DO
$$
    DECLARE

        cmd TEXT;

    BEGIN
        SELECT
            'SELECT (' ||
            STRING_AGG('(SELECT setval(''' || s.sequencename::REGCLASS::TEXT || ''', max(id), true) FROM ' ||
                       i.schemaname :: TEXT || '.' || i.tablename :: TEXT ||
                ')', ', ') ||
            ')'
        INTO cmd
        FROM
            pg_tables i
        JOIN pg_sequences s
            ON s.schemaname = i.schemaname AND s.sequencename = (i.tablename :: TEXT || '_id_seq')
        WHERE
            s.sequenceowner :: TEXT = '[ROLE]' -- 'postgres'
            AND NULLIF(s.last_value, 0) IS NOT NULL;

        IF ( cmd IS NOT NULL ) THEN
            EXECUTE cmd;
        END IF;
    END;
$$;
```

## Find the sequences not attached to ID columns
*(for manual adjustment where needed)*
```sql
SELECT *
FROM
    pg_sequences s
WHERE
    NOT EXISTS (
        SELECT 1
        FROM pg_tables i
        WHERE s.sequencename = ( i.tablename :: TEXT || '_id_seq' )
    )
    AND s.sequenceowner :: TEXT = '[ROLE]' -- 'postgres'
    AND NULLIF(s.last_value, 0) IS NOT NULL;
```

## Dump database on remote host to file
```bash
$ pg_dump -U username -h hostname databasename > dump.sql
```

## Import dump into existing database
```bash
$ psql -d newdb -f dump.sql
```

## Get locked queries
```sql
  SELECT blocked_locks.pid     AS blocked_pid,
         blocked_activity.usename  AS blocked_user,
         blocking_locks.pid     AS blocking_pid,
         blocking_activity.usename AS blocking_user,
        -- trim(blocked_activity.query)    AS blocked_statement,
         trim(regexp_replace(blocked_activity.query, E'[\\r\\n ]+', ' ', 'g' )) AS blocked_statement,
       --trim(blocking_activity.query)   AS current_statement_in_blocking_process
         trim(regexp_replace(blocking_activity.query, E'[\\r\\n ]+', ' ', 'g' )) AS current_statement_in_statement  
   FROM  pg_catalog.pg_locks         blocked_locks
    JOIN pg_catalog.pg_stat_activity blocked_activity  ON blocked_activity.pid = blocked_locks.pid
    JOIN pg_catalog.pg_locks         blocking_locks 
        ON blocking_locks.locktype = blocked_locks.locktype
        AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
        AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
        AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
        AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
        AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
        AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
        AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
        AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
        AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
        AND blocking_locks.pid != blocked_locks.pid
    JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
    WHERE NOT blocked_locks.granted;
```

## Get long-running queries
```sql
SELECT
    now() - query_start                                  as duration,
    trim(regexp_replace(query, E'[\\r\\n ]+', ' ', 'g')) as sql_query,
    application_name,
    *
FROM
    pg_stat_activity
WHERE
    state <> 'idle'
--         AND pg_stat_activity.datname = 'DATABASE_NAME'
--         AND application_name in ( 'APPLICATION_NAME' )
    AND now() - query_start > '1 minutes'::INTERVAL
    AND pid <> pg_backend_pid()
ORDER BY
    1 DESC;
```

## Get config file changes vs defaults
*courtesy of Amir M*
```sql
SELECT
    name
  , CASE setting
      WHEN '0'  THEN setting
      WHEN '-1' THEN setting
      ELSE
        CASE unit
          WHEN '16MB' THEN pg_size_pretty(setting::numeric * 16 * 1024 * 1024)
          WHEN '8kB'  THEN pg_size_pretty(setting::numeric * 8 * 1024)
          WHEN 'kB'   THEN pg_size_pretty(setting::numeric * 1024)
          ELSE             setting || '' || coalesce(unit, '')
        END
    END
    AS current_setting
  --, setting || ' ' || coalesce(unit, '') AS current_setting_raw
  , CASE boot_val
      WHEN '0'  THEN boot_val
      WHEN '-1' THEN boot_val
      ELSE
        CASE unit
          WHEN '16MB' THEN pg_size_pretty(boot_val::numeric * 16 * 1024 * 1024)
          WHEN '8kB'  THEN pg_size_pretty(boot_val::numeric * 8 * 1024)
          WHEN 'kB'   THEN pg_size_pretty(boot_val::numeric * 1024)
          ELSE             boot_val || '' || coalesce(unit, '')
        END
    END AS default_setting
  --, boot_val || ' ' || coalesce(unit, '') AS default_setting_raw
  , category
  --, source
  --, sourcefile
  --, sourceline
  , CASE source
      WHEN 'configuration file' THEN substring(sourcefile, length(sourcefile) - position('/' in (reverse(sourcefile))) + 2) || ':' || sourceline
      ELSE sourcefile
    END AS sourcefile
FROM
  pg_settings
WHERE
  setting <> boot_val AND source = 'configuration file'
ORDER BY
  category DESC, name;
```

## First, last, and first/last aggs
*Not sure where these came from, pretty sure was PG wiki*
```sql
-- Create a function that always returns the first non-NULL value:
CREATE OR REPLACE FUNCTION public.first_agg (anyelement, anyelement)
  RETURNS anyelement
  LANGUAGE sql IMMUTABLE STRICT PARALLEL SAFE AS
'SELECT $1';

-- Then wrap an aggregate around it:
CREATE AGGREGATE public.first (anyelement) (
  SFUNC    = public.first_agg
, STYPE    = anyelement
, PARALLEL = safe
);

-- Create a function that always returns the last non-NULL value:
CREATE OR REPLACE FUNCTION public.last_agg (anyelement, anyelement)
  RETURNS anyelement
  LANGUAGE sql IMMUTABLE STRICT PARALLEL SAFE AS
'SELECT $2';

-- Then wrap an aggregate around it:
CREATE AGGREGATE public.last (anyelement) (
  SFUNC    = public.last_agg
, STYPE    = anyelement
, PARALLEL = safe
);
```
