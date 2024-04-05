---
title: Migrate to TSDB
menuTitle: Migrate to TSDB
description: Migration guide for moving from any of the older indexes to TSDB
weight: 100
keywords:
  - migrate
  - tsdb
---

# Migrate to TSDB

[TSDB]({{< relref "../../../operations/storage/tsdb" >}}) is the recommended index type for Loki and is where the current development lies.
If you are running Loki with [boltb-shipper]({{< relref "../../../operations/storage/boltdb-shipper" >}}) or any of the [legacy index types](https://grafana.com/docs/loki/latest/configure/storage/#index-storage) that have been deprecated,
we strongly recommend migrating to TSDB.


### Configure TSDB index for an upcoming period

To begin the migration, add a new [period_config]({{< relref "../../../configure#period_config" >}}) entry in your [schema_config]({{< relref "../../../configure#schema_config" >}}).
You can read more about schema config [here](https://grafana.com/docs/loki/latest/configure/storage/#schema-config).

{{% admonition type="note" %}}
You must roll out the new `period_config` change to all Loki components in order for it to take effect.
{{% /admonition %}}

This example adds a new `period_config` which configures Loki to start using the TSDB index for the data ingested starting from `2023-10-20`.

```
schema_config:
  configs:
    - from: 2023-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h
    - from: 2023-10-20 ①
      store: tsdb ②
      object_store: filesystem ③
      schema: v13 ④
      index:
        prefix: index_
        period: 24h
```

①  You must set the new period `from` to a date in the future.

②  Update the new period to use TSDB as the index type by setting `store: tsdb`.

③  This sample configuration uses filesystem as the storage in both the periods. If you want to use a different storage for the TSDB index and chunks, you can specify a different `object_store` in the new period.

④  Update the schema to v13 which is the recommended version at the time of writing. Please refer to the [configure page]({{< relref "../../../configure#period_config" >}}) for the current recommend version.

### Configure TSDB shipper

It's also important that you configure the `tsdb_shipper` block in [storage_config]({{< relref "../../../configure#storage_config" >}}). Specifically the following options:
- `active_index_directory`: directory where ingesters would write index files which will then be uploaded by shipper to configured storage.
- `cache_location`: cache location for downloading index files from the storage for use in query path.

```
storage_config:
  tsdb_shipper:
    active_index_directory: /data/tsdb-index
    cache_location: /data/tsdb-cache
```

### Run compactor

We strongly recommended running the [compactor]({{< relref "../../../operations/storage/retention#compactor" >}}) when using TSDB index. It is responsible for running compaction and retention on TSDB index.
Not running index compaction will result in sub-optimal query performance.

Please refer to the [compactor section]({{< relref "../../../operations/storage/retention#compactor" >}}) for more information and configuration examples.