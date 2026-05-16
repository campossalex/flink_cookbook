# Self-managed PostgreSQL CDC

## Edit postgresql.conf:

```
wal_level = logical
max_wal_senders = 10
max_replication_slots = 10
```
Restart the PostgreSQL server   

## Set REPLICA IDENTITY FULL on Each Table

For Debezium (which powers the connector) to capture the full before-image on UPDATE and DELETE events, each table you want to capture must have REPLICA IDENTITY FULL:


```
ALTER TABLE my_table REPLICA IDENTITY FULL;
```
Without this, updates/deletes may not include all columns

## Create a Publication

Connect to your database as a superuser and create a publication for the tables you want to capture:

-- For specific tables:
```
CREATE PUBLICATION my_publication FOR TABLE my_table, my_other_table;
```
-- Or for ALL tables:
```
CREATE PUBLICATION all_tables_pub FOR ALL TABLES;
```

## (Optional) Create a Logical Replication Slot

If your CDC tool does not create slots automatically, create one manually:

```
SELECT * FROM pg_create_logical_replication_slot('flink_cdc_slot', 'pgoutput');
```

## Verify the Setup
```
-- Check WAL level
SHOW wal_level;          -- should return "logical"

-- List publications
SELECT * FROM pg_publication;

-- List replication slots
SELECT * FROM pg_replication_slots;

```


## Table DDL:

```
CREATE TABLE master_product (
  product_id string,
  category string,
  item string,
  size string,
  cogs string,
  price string,
  inventory_level string,
  contains_fruit string,
  contains_veggies string,
  contains_nuts string,
  contains_caffeine string,
  PRIMARY KEY (product_id) NOT ENFORCED
) WITH (
  'connector'                 = 'postgres-cdc',
  'hostname'                  = 'host.minikube.internal',
  'port'                      = '5432',
  'username'                  = '<user>',
  'password'                  = '<password>',
  'database-name'             = 'warehouse',
  'schema-name'               = 'public',
  'table-name'                = 'master_product',
  'slot.name'                 = 'flink_cdc_slot',
  'decoding.plugin.name'      = 'pgoutput',
  'debezium.publication.name' = 'all_tables_pub'
);
```
