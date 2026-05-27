# Fluss and Flink

## Create Fluss catalog

```
CREATE CATALOG fluss WITH (
  'type' = 'fluss',
  'bootstrap.servers' = 'coordinator-server-0.coordinator-server-hs.fluss.svc.cluster.local:9124'
  );
```

## Create Fluss database

```
CREATE DATABASE fluss.app;
```

## Create Fluss table

```
CREATE TABLE IF NOT EXISTS fluss.app.pk_table (
    id          BIGINT,
    name        STRING,
    `date`      STRING,
    age         INT,
    score       DOUBLE,
    active      BOOLEAN,
    balance     DOUBLE,
    PRIMARY KEY (id, `date`) NOT ENFORCED
) PARTITIONED BY (`date`) WITH (
    'bucket.num'                          = '3',
    'table.auto-partition.enabled'        = 'true',
    'table.auto-partition.time-unit'      = 'DAY',
    'table.auto-partition.num-precreate'  = '7',
    'table.auto-partition.num-retention'  = '10',
    'table.auto-partition.time-zone'      = 'Europe/Berlin'
);
````

## Create a Faker to Fluss job

```
CREATE TEMPORARY TABLE fake WITH (
    'connector'                  = 'faker',
    'rows-per-second'            = '2000',
    'fields.id.expression'       = '#{number.numberBetween ''1'',''100000''}',
    'fields.name.expression'     = '#{superhero.name}',
    'fields.date.expression'     = '202605#{regexify ''(01|02|03|04|05|06|07)''}',
    'fields.age.expression'      = '#{number.numberBetween ''18'',''80''}',
    'fields.score.expression'    = '#{number.randomDouble ''2'',''0'',''100''}',
    'fields.active.expression'   = '#{regexify ''(true|false){1}''}',
    'fields.balance.expression'  = '#{number.numberBetween ''0'',''1000000''}'
) LIKE fluss.app.pk_table (EXCLUDING ALL);

INSERT INTO fluss.app.pk_table SELECT * FROM fake;
```
