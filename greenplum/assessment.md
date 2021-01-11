# GreenPlum Assessment Scripts

### Version:
```sql
select
    version();
```

### Database and its size:
```sql
select
    sodddatname as database,
    cast(sodddatsize as float)/ 1024 / 1024 / 1024 as size_in_gb
from
    gp_size_of_database;
```

### Schema size:
```sql
select
    sosdnsp as schema,
    cast(sosdschematablesize as float)/ 1024 / 1024 / 1024 as table_size_gb,
    cast(sosdschemaidxsize as float)/ 1024 / 1024 / 1024 as index_size_gb
from
    gp_size_of_schema_disk
order by
    sosdnsp ;
```

### Check table size:
```sql
select
    sotdschemaname,
    relname as name,
    cast(sotdsize as float)/ 1024 / 1024 / 1024 as size,
    sotdtoastsize as toast,
    sotdadditionalsize as other
from
    gp_size_of_table_disk as sotd,
    pg_class
where
    sotd.sotdoid = pg_class.oid
order by
    relname;
```

### Check uncompressed size:
```sql
select
    sotuschemaname,
    sotutablename,
    cast(sotusize as float)/ 1024 / 1024 / 1024
from
    gp_size_of_table_uncompressed;
```

### Compress and Uncomoress size:
```sql
WITH unc 
     AS (SELECT sotuschemaname, 
                sotutablename, 
                Cast(sotusize AS FLOAT) / 1024 / 1024 / 1024 AS SIZE 
         FROM   gp_size_of_table_uncompressed), 
     comp 
     AS (SELECT sotdschemaname, 
                sotdtablename, 
                Cast(sotdsize AS FLOAT) / 1024 / 1024 / 1024 AS SIZE, 
                sotdtoastsize                                AS toast, 
                sotdadditionalsize                           AS other 
         FROM   gp_size_of_table_disk) 
SELECT a.sotuschemaname, 
       a.sotutablename, 
       a.SIZE AS uncompressed, 
       b.SIZE AS compressed, 
       b.toast, 
       b.other 
FROM   unc a 
       join comp b 
         ON a.sotuschemaname = b.sotdschemaname 
            AND a.sotutablename = b.sotdtablename 
ORDER  BY 1, 
          2; 
```          

### Partition size:
```sql
select
    Sopaidparentschemaname as parent_schema,
    Sopaidparenttablename as parent_table,
    Sopaidpartitionschemaname as partition_schema,
    sopaidpartitiontablename as partition_table,
    cast(sopaidpartitiontablesize as float)/ 1024 / 1024 / 1024 size_in_GB,
    sopaidpartitionindexessize as total_index_size
from
    gp_size_of_partition_and_indexes_disk
order by
    1,
    2,
    3,
    4;
```

### Data Types:
```sql
select
    distinct(data_type) as datatype
from
    information_schema.columns
where
    table_schema not in (
        'gp_toolkit', 'information_schema', 'pg_aoseg', 'pg_bitmapindex', 'pg_catalog'
    );
```    

###Extended data types:
```sql
with cte as (
    select
        distinct table_schema, data_type as data_type
    from
        information_schema.columns
    where
        table_schema not in (
            'gp_toolkit', 'information_schema', 'pg_aoseg', 'pg_bitmapindex', 'pg_catalog'
        )
)
select
    table_schema,
    string_agg(data_type, ', ') as data_type
from
    cte
group by
    table_schema;
```

### Table Type:
```sql
select
    table_catalog,
    table_schema,
    SUM(case when table_type = 'BASE TABLE' then 1 else 0 end) as BASE_TABLE,
    SUM(case when table_type = 'VIEW' then 1 else 0 end) as view
from
    information_schema.tables
group by
    1,
    2
    order by 1,2;
```

### Find Partitions:
```sql
select
    schemaname,
    tablename,
    partitiontype
from
    pg_partitions
group by
    schemaname,
    tablename,
    partitiontype
order by
    1,
    2;
```

### Extended - find partitions:
```sql
select
    pt.schemaname,
    pt.tablename,
    ptc.columnname ,
    pt.partitiontype,
    case
        when pt.partitionboundary like 'PARTITION%'
        and pt.partitiontype = 'list' then 'Main Partition'
        when pt.partitionboundary like 'SUB%'
        and pt.partitiontype = 'list' then 'SUB Partition'
        when (
            pt.partitionboundary not like('PARTITION%')
            or pt.partitionboundary not like 'SUB%'
        )
        and pt.partitiontype = 'list' then 'Other Parition'
        when pt.partitiontype = 'range' then 'Main Partition'
    end as paritionclass
from
    pg_partitions pt
join pg_catalog.pg_partition_columns ptc on
    pt.schemaname = ptc.schemaname
    and pt.tablename = ptc.tablename
    and pt.partitionlevel = ptc.partitionlevel
group by
    pt.schemaname,
    pt.tablename,
    ptc.columnname ,
    pt.partitiontype,
    paritionclass
order by
    1,
    2;
```    

### Partition Columns:
```sql
select
    *
from
    pg_catalog.pg_partition_columns;
```

### Table Rows:
```sql
SELECT ns.nspname            AS SCHEMA, 
       c.relname             AS TABLE, 
       c.reltuples :: bigint AS ROWS 
FROM   pg_catalog.pg_class AS c 
       join pg_catalog.pg_namespace AS ns 
         ON c.relnamespace = ns.oid 
WHERE  c.relkind = 'r' 
       AND ns.nspname NOT LIKE 'pg_%' 
       AND ns.nspname NOT IN( 'gp_toolkit', 'gptext', 'information_schema' ) 
       AND relstorage != 'x' 
ORDER  BY 1, 
          2; 
```

### External table:
```sql
SELECT ns.nspname  SCHEMA, 
       c.relname   AS Table, 
       ex.location AS path, 
       CASE 
         WHEN ex.fmttype = 't' THEN 'text' 
         WHEN ex.fmttype = 'c' THEN 'csv' 
         ELSE 'Unknown' 
       END         AS format, 
       CASE 
         WHEN ex.writable = '0' THEN 'Yes' 
         ELSE 'No' 
       END         AS is_readonly 
FROM   pg_exttable ex 
       join pg_class c 
         ON ex.reloid = c.oid 
       left outer join pg_catalog.pg_namespace ns 
                    ON c.relnamespace = ns.oid 
WHERE  ns.nspname NOT LIKE 'pg_%' 
       AND ns.nspname NOT IN( 'gp_toolkit', 'gptext', 'information_schema' ) 
ORDER  BY 1, 
          2; 
```          
