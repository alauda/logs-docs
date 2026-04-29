# ClickHouse Backup and Restore on Local Storage or NAS

## Overview

This document describes how to back up and restore ClickHouse tables in the `observability` database by using local storage or a NAS-mounted directory. This procedure applies to clusters that use the `ReplicatedMergeTree` table engine.

For ClickHouse, a NAS mount is treated as a local filesystem path. Therefore, you can use the same backup and restore method for both local storage and NAS.

This document provides the following guidance:

- Create a full backup.
- Create an incremental backup based on a full backup.
- Store backup data in a local directory or NAS mount path.
- Restore data from a local or NAS backup.
- Validate backup and restore results.

This document uses the `observability.audit` table as an example. You can apply the same procedure to other tables.

The following tables are common examples:

- `audit`: stores audit data.
- `event`: stores event data.
- `log_kubernetes`: stores Kubernetes logs.
- `log_platform`: stores platform service logs.
- `log_system`: stores node-level system logs.
- `log_workload`: stores application and workload logs.

The storage types differ as follows:

- Local storage: backups are written to a node-local directory, such as `/var/lib/clickhouse/backup/` or `/data/clickhouse-backup/`.
- NAS: backups are written to a mounted shared storage path, such as `/mnt/nas/clickhouse-backup/` or `/backup/clickhouse/`.

NAS is more suitable for centralized management and cross-node recovery. Local storage is usually simpler to configure and often provides better performance.

## Prerequisites

Before you start, make sure the following conditions are met.

### Environment Requirements

| Item | Requirement |
| --- | --- |
| ClickHouse | Version 25.3 or later |
| Table engine | ReplicatedMergeTree |
| Backup target | Local directory or NAS directory mounted to the host |

### Access Requirements

All SQL statements in this document use the built-in ClickHouse administrator account `default`.

You can run the SQL statements on any healthy ClickHouse instance. For consistency, this document uses a single ClickHouse Pod as an example.

Before you run SQL statements, connect to the target Pod:

```bash
kubectl exec -ti -n cpaas-system chi-cpaas-clickhouse-replicated-0-0-0 -- bash
```

Then connect to ClickHouse in the container:

```bash
clickhouse-client
```

The `default` user already has the required privileges for this procedure, including `BACKUP`, `RESTORE`, `SELECT`, and `ALTER`.

### Directory Requirements

Make sure the following conditions are met:

- The ClickHouse process has read and write access to the target directory.
- The target directory has sufficient available capacity.
- If NAS is used, the mount point is restored automatically after Pod or host restarts.
- In Kubernetes, the directory is mounted through PVC, hostPath, or CSI.

### Backup Strategy

Use the following backup strategy:

- Run the backup operation only once on any healthy replica.
- Use `BACKUP TABLE` to create a consistent snapshot without stopping the service.
- Use `base_backup` for file-level deduplication in incremental backups.
- Keep the base full backup accessible when you restore an incremental backup.
- Use one full backup per week and one incremental backup per day to balance restore complexity and storage cost.

## Procedure

The following examples use an initial full backup and a daily incremental backup.

### Create a Full Backup

A full backup creates the baseline for subsequent incremental backups.

Run the following command on any healthy ClickHouse instance:

```sql
BACKUP TABLE observability.audit
TO File('audit_full_20260423')
SETTINGS compression_method = 'zstd';
```

Notes:

- `File(...)` writes the backup to a local path under the ClickHouse backup directory.
- `compression_method = 'zstd'` compresses the backup content with zstd to reduce storage usage.
- If you need to archive the backup to NAS or another directory, copy the backup after the backup task is complete.

### Validate Backup Success

After the backup is complete, validate the result.

#### Check Backup Task Status

Run the following query on the ClickHouse instance where the backup command was executed:

```sql
SELECT
    id,
    name,
    status,
    error,
    start_time,
    end_time,
    num_files,
    total_size
FROM system.backups
ORDER BY start_time DESC
LIMIT 10
FORMAT Vertical;
```

Expected result:

Locate the backup record for `observability.audit` through the `name` field. The backup is successful if the following conditions are met:

- `status = 'BACKUP_CREATED'`
- `error` is empty
- `end_time` has a value
- `num_files > 0`
- `total_size > 0`

#### Check the Backup Directory

Run the following command on the node or in the container where the backup directory is available:

```bash
ls -lah /cpaas/data/clickhouse/backups
```

Expected result:

- The backup directory exists.
- The directory contains files or folders generated by this backup.
- The file count and total size are greater than 0.

### Create an Incremental Backup

An incremental backup uses `base_backup` and writes only new or changed data files.

Run the following command on any healthy ClickHouse instance:

```sql
BACKUP TABLE observability.audit
TO File('audit_incr_20260424')
SETTINGS
  base_backup = File('audit_full_20260423'),
  compression_method = 'zstd';
```

Notes:

- The incremental backup depends on the backup specified by `base_backup`.
- This document recommends that each daily incremental backup uses the latest full backup as its `base_backup`, instead of using the previous incremental backup as the base. This keeps the restore dependency simple: restoring a daily incremental backup only requires the incremental backup and its corresponding full backup.
- Keep the dependent full backup when you restore the incremental backup.
- Create a new full backup periodically to avoid relying on the same baseline for too long.

### Archive the Backup Files

If you need to archive the backup files to a designated local directory or a NAS mount path, copy the files after the backup succeeds.

Run the following commands on the node where the ClickHouse backup directory is available:

```bash
cp -r /cpaas/data/clickhouse/backups/audit_full_20260423 /my_dir/audit_full_20260423
cp -r /cpaas/data/clickhouse/backups/audit_incr_20260424 /my_dir/audit_incr_20260424
```

Notes:

- After each full or incremental backup, copy the corresponding backup directory to the archive location.
- After the incremental backup is copied successfully, you can delete the local incremental backup if it is no longer needed.
- Delete the local full backup only after the next full backup succeeds and the retention policy allows cleanup.

## Restore

Use this procedure when table data is corrupted, the data directory is deleted, or the table state is abnormal.

### Restore Prerequisites

Before you start the restore procedure, make sure the following conditions are met:

- You have a valid full backup or incremental backup.
- If you restore from an incremental backup, the corresponding full backup is available in the restore path.
- You know the archive directory that stores the latest incremental backup and the corresponding full backup.
- You have access to the Kubernetes cluster and the host nodes where ClickHouse instances are scheduled.

### Restore Procedure

This procedure uses a unified recovery method: stop related components, clean the local ClickHouse data directory on each ClickHouse host node, start ClickHouse only, copy the backup files to the ClickHouse backup directory, restore data from the local files, and then start the related components.

In this deployment, ClickHouse Keeper is integrated with ClickHouse and its data is also stored under the ClickHouse data directory. Therefore, cleaning `/cpaas/data/clickhouse/*` also removes the local ClickHouse Keeper data.

#### Stop Related Components and Clean the Data Directory

> Warning:
> Cleaning the data directory deletes local ClickHouse data on the target nodes. Confirm that the backup is available before you continue.
>
> Run the `rm -rf /cpaas/data/clickhouse/*` command on each ClickHouse host node, not inside the ClickHouse container.

Stop the `clickhouse-operator`, `sentry`, `razor`, and ClickHouse components:

```bash
kubectl scale deploy -n cpaas-logging clickhouse-operator --replicas=0
kubectl scale deploy -n cpaas-system sentry --replicas=0
kubectl scale sts -n cpaas-system razor --replicas=0
kubectl scale sts -n cpaas-system chi-cpaas-clickhouse-replicated-0-0 --replicas=0
kubectl scale sts -n cpaas-system chi-cpaas-clickhouse-replicated-0-1 --replicas=0
kubectl scale sts -n cpaas-system chi-cpaas-clickhouse-replicated-0-2 --replicas=0
```

Confirm that all related Pods have stopped:

```bash
kubectl get pod -A | grep -E "razor-0|razor-1|clickhouse|sentry" | grep -v sentry-service
```

On each host node where a ClickHouse instance is deployed, clean the local ClickHouse data directory:

```bash
rm -rf /cpaas/data/clickhouse/*
```

#### Start ClickHouse Only

Start only the ClickHouse components. Do not start `clickhouse-operator`, `sentry`, or `razor` yet.

```bash
kubectl scale sts -n cpaas-system chi-cpaas-clickhouse-replicated-0-0 --replicas=1
kubectl scale sts -n cpaas-system chi-cpaas-clickhouse-replicated-0-1 --replicas=1
kubectl scale sts -n cpaas-system chi-cpaas-clickhouse-replicated-0-2 --replicas=1
```

Confirm that all ClickHouse Pods are running:

```bash
kubectl get pod -A | grep -i "chi-cpaas-clickhouse-replicated"
```

#### Copy Backup Files to the Restore Path

Copy the latest incremental backup and the corresponding full backup from the archive directory to the ClickHouse backup directory on any ClickHouse host node.

Run the following commands on the selected ClickHouse host node:

```bash
mkdir -p /cpaas/data/clickhouse/backups
cp -r /my_dir/audit_full_20260423 /cpaas/data/clickhouse/backups/audit_full_20260423
cp -r /my_dir/audit_incr_20260424 /cpaas/data/clickhouse/backups/audit_incr_20260424
```

Notes:

- If you restore from an incremental backup, prepare the dependent full backup at the same time.
- The backup files must be placed under the ClickHouse backup directory so that `RESTORE ... FROM File(...)` can access them.
- Replace `/my_dir`, `audit_full_20260423`, and `audit_incr_20260424` with the actual archive path and backup directory names.

#### Run the Restore Command

Run the following command on any healthy ClickHouse instance.

For a `ReplicatedMergeTree` table in this 3-replica deployment, use `ON CLUSTER 'replicated'` so that the restore operation is distributed to all ClickHouse replicas in the cluster.

```sql
RESTORE TABLE observability.audit
ON CLUSTER 'replicated'
FROM File('audit_incr_20260424');
```

Notes:

- If you restore from an incremental backup, ClickHouse reads the dependent base backup automatically.
- Keep the corresponding full backup directory accessible during the restore.
- Replace `observability.audit` and the backup directory name with the actual table and backup that you want to restore.
- The table list is determined by the customer. This document uses `observability.audit` only as an example.

#### Start the Related Components

After the restore is complete, start the `clickhouse-operator`, `sentry`, and `razor` components again:

```bash
kubectl scale deploy -n cpaas-logging clickhouse-operator --replicas=1
kubectl scale deploy -n cpaas-system sentry --replicas=2
kubectl scale sts -n cpaas-system razor --replicas=2
```

## Validate Restore Success

After the restore is complete, validate the result from the data, partition, and replica perspectives.

### Validate Total Row Count

Run the following query on each ClickHouse instance:

```sql
SELECT count(*) AS total_rows
FROM observability.audit;
```

Expected result:

The `total_rows` value is identical on all ClickHouse instances.

### Validate Partition-Level Data

Run the following query on each ClickHouse instance:

```sql
SELECT
    partition,
    sum(rows) AS rows,
    count() AS active_parts
FROM system.parts
WHERE active
  AND database = 'observability'
  AND table = 'audit'
GROUP BY partition
ORDER BY partition;
```

Expected result:

- The `partition` list is identical on all ClickHouse instances.
- The `rows` value for each partition is identical on all ClickHouse instances.

`active_parts` is only used to observe the physical part layout. A different number of parts does not necessarily indicate a restore failure.

### Validate Replica Status

Run the following query on any ClickHouse instance:

```sql
SELECT
    database,
    table,
    total_replicas,
    active_replicas,
    queue_size,
    absolute_delay
FROM system.replicas
WHERE database = 'observability'
  AND table = 'audit';
```

Expected result:

For a 3-replica cluster, the restore is successful if the following conditions are met:

- `total_replicas = 3`
- `active_replicas = 3`
- `queue_size = 0`
- `absolute_delay` is close to `0`

## Recommendations

Use the following recommendations in production:

- Maintain the backup chain with one full backup per week and one incremental backup per day.
- Use a consistent naming convention for backup directories, such as `full_YYYYMMDD` and `incr_YYYYMMDD`.
- Run restore drills in a test environment on a regular basis.
- Define a retention policy for expired incremental backups and historical full backups.
