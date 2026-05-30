# Postgres CDC to Fluss PK Table

## Edit postgresql.conf:

```
wal_level = logical
max_wal_senders = 10
max_replication_slots = 10
```
Restart the PostgreSQL server   

## Create the source table and user

```
CREATE DATABASE warehouse;

\c warehouse;

CREATE TABLE master_product (
  product_id        VARCHAR(64)   NOT NULL,
  category          VARCHAR(128),
  item              VARCHAR(255),
  size              VARCHAR(32),
  cogs              NUMERIC(10, 2),
  price             NUMERIC(10, 2),
  inventory_level   INTEGER,
  contains_fruit    BOOLEAN,
  contains_veggies  BOOLEAN,
  contains_nuts     BOOLEAN,
  contains_caffeine BOOLEAN,
  PRIMARY KEY (product_id)
);

CREATE ROLE cdc_user WITH LOGIN REPLICATION PASSWORD 'admin123';

GRANT CONNECT  ON DATABASE warehouse TO cdc_user;
GRANT USAGE    ON SCHEMA   public    TO cdc_user;
GRANT SELECT   ON ALL TABLES IN SCHEMA public TO cdc_user;

ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO cdc_user;

ALTER ROLE cdc_user WITH REPLICATION;

INSERT INTO master_product
  (product_id, category, item, size, cogs, price, inventory_level,
   contains_fruit, contains_veggies, contains_nuts, contains_caffeine)
VALUES
  ('CS01', 'Classic Smoothies',      'Sunrise Sunset',     '24 oz.', 1.50, 4.99, 75, true,  false, false, false),
  ('CS02', 'Classic Smoothies',      'Kiwi Quencher',      '24 oz.', 1.50, 4.99, 75, true,  false, false, false),
  ('CS03', 'Classic Smoothies',      'Paradise Point',     '24 oz.', 1.50, 4.99, 75, true,  false, false, false),
  ('CS04', 'Classic Smoothies',      'Sunny Day',          '24 oz.', 1.50, 4.99, 75, true,  false, false, false),
  ('CS05', 'Classic Smoothies',      'Mango Magic',        '24 oz.', 1.50, 4.99, 75, true,  false, false, false),
  ('CS06', 'Classic Smoothies',      'Blimey Limey',       '24 oz.', 1.50, 4.99, 75, true,  false, false, false),
  ('CS07', 'Classic Smoothies',      'Blueberry Bliss',    '24 oz.', 1.50, 4.99, 75, true,  false, false, false),
  ('CS08', 'Classic Smoothies',      'Rockin'' Raspberry', '24 oz.', 1.50, 4.99, 75, true,  false, false, false),
  ('CS09', 'Classic Smoothies',      'Strawberry Limeade', '24 oz.', 1.50, 4.99, 75, true,  false, false, false),
  ('CS10', 'Classic Smoothies',      'Peaches ''n Silk',   '24 oz.', 1.50, 4.99, 75, true,  false, false, false),
  ('CS11', 'Classic Smoothies',      'Jetty Punch',        '24 oz.', 1.50, 4.99, 75, true,  false, false, false),
  ('SF01', 'Superfoods Smoothies',   'Island Green',       '24 oz.', 2.10, 5.99, 50, true,  true,  false, false),
  ('SF02', 'Superfoods Smoothies',   'Totally Green',      '24 oz.', 2.10, 5.99, 50, true,  true,  false, false),
  ('SF03', 'Superfoods Smoothies',   'Acai Berry Boost',   '24 oz.', 2.10, 5.99, 50, true,  false, false, false),
  ('SF04', 'Superfoods Smoothies',   'Pomegranate Plunge', '24 oz.', 2.10, 5.99, 50, true,  false, false, false),
  ('SF05', 'Superfoods Smoothies',   'Caribbean C-Burst',  '24 oz.', 2.10, 5.99, 50, true,  false, false, false),
  ('SF06', 'Superfoods Smoothies',   'Get Up and Goji',    '24 oz.', 2.10, 5.99, 50, true,  true,  false, false),
  ('SF07', 'Superfoods Smoothies',   'Detox Island Green', '24 oz.', 2.10, 5.99, 50, true,  true,  false, false),
  ('SC01', 'Supercharged Smoothies', 'Triple Berry Oat',   '24 oz.', 2.70, 5.99, 35, true,  false, false, false),
  ('SC02', 'Supercharged Smoothies', 'Peanut Paradise',    '24 oz.', 2.70, 5.99, 35, false, false, false, false),
  ('SC03', 'Supercharged Smoothies', 'Health Nut',         '24 oz.', 2.70, 5.99, 35, false, false, true,  false),
  ('SC04', 'Supercharged Smoothies', 'Lean Machine',       '24 oz.', 2.70, 5.99, 35, false, true,  true,  true),
  ('SC05', 'Supercharged Smoothies', 'Muscle Blaster',     '24 oz.', 2.70, 5.99, 35, false, false, true,  true),
  ('IS01', 'Indulgent Smoothies',    'Bahama Mama',        '24 oz.', 2.20, 5.49, 60, true,  false, false, false),
  ('IS02', 'Indulgent Smoothies',    'Peanut Butter Cup',  '24 oz.', 2.20, 5.49, 60, false, false, true,  false),
  ('IS03', 'Indulgent Smoothies',    'Beach Bum',          '24 oz.', 2.20, 5.49, 60, true,  true,  false, false),
  ('IS04', 'Indulgent Smoothies',    'Mocha Madness',      '24 oz.', 2.20, 5.49, 60, false, false, true,  true);`
```

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


## Source Table DDL (postgres-cdc):

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

## Sink Table DDL (fluss):

```
CREATE CATALOG fluss WITH (
  'type' = 'fluss',
  'bootstrap.servers' = 'coordinator-server-0.coordinator-server-hs.fluss.svc.cluster.local:9124'
  );

CREATE DATABASE fluss.app;

CREATE TABLE IF NOT EXISTS fluss.app.product_table (
  product_id STRING,
  category STRING,
  item STRING,
  size STRING,
  cogs STRING,
  price STRING,
  inventory_level STRING,
  contains_fruit STRING,
  contains_veggies STRING,
  contains_nuts STRING,
  contains_caffeine STRING,
  PRIMARY KEY (product_id) NOT ENFORCED
) WITH (
    'bucket.num' = '3'
);
```

## Source-to-Sink Job

Straightforward data ingestion (no transformations)

```
INSERT INTO fluss.app.product_table
SELECT * FROM master_product;
```
