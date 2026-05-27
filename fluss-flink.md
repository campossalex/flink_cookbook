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
