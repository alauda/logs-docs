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

- LocalVolume: ClickHouse data is stored in a node-local directory, such as `/cpaas/data/clickhouse/`. `BACKUP ... TO File(...)` writes backup files to the ClickHouse backup directory under this local data directory, such as `/cpaas/data/clickhouse/backups`.
- StorageClass, such as TopoLVM, NFS, or Ceph: ClickHouse data is stored on the corresponding StorageClass volume. `BACKUP ... TO File(...)` writes backup files to the ClickHouse backup directory on that volume.
- NAS archive path: backup files can be copied from the ClickHouse backup directory to a mounted NAS path or another custom backup directory for long-term retention.

For both LocalVolume and StorageClass deployments, use the actual ClickHouse backup directory as the source path when archiving backup files and as the destination path when copying backup files back for restore.

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

- `File(...)` writes the backup to the ClickHouse backup directory on the ClickHouse data volume.
- For LocalVolume deployments, the backup directory is typically `/cpaas/data/clickhouse/backups` on the host node.
- For StorageClass deployments, such as TopoLVM, NFS, or Ceph, the backup directory is on the corresponding StorageClass volume.
- `compression_method = 'zstd'` compresses the backup content with zstd to reduce storage usage.
- If you need to archive the backup to NAS or another custom backup directory, copy the backup after the backup task is complete.

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

Run the following command on the node or in the container where the ClickHouse backup directory is available.

For LocalVolume deployments, the backup directory is typically `/cpaas/data/clickhouse/backups`:

```bash
ls -lah /cpaas/data/clickhouse/backups
```

For StorageClass deployments, check the backup directory on the corresponding StorageClass volume.

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

`BACKUP ... TO File(...)` writes the backup files to the ClickHouse backup directory on the ClickHouse data volume where the backup command is executed.

The backup directory depends on the storage type:

- For LocalVolume deployments, the ClickHouse data directory is on the host node, typically `/cpaas/data/clickhouse/`, and the backup files are generated under `/cpaas/data/clickhouse/backups`.
- For StorageClass deployments, such as TopoLVM, NFS, or Ceph, the ClickHouse data directory is on the corresponding StorageClass volume, and the backup files are generated under the ClickHouse backup directory on that volume.

After the backup succeeds, copy the backup files from the actual ClickHouse backup directory to a custom backup directory or NAS mount path for retention.

For LocalVolume deployments, the source path is typically `/cpaas/data/clickhouse/backups`:

```bash
cp -r /cpaas/data/clickhouse/backups/audit_full_20260423 /my_dir/audit_full_20260423
cp -r /cpaas/data/clickhouse/backups/audit_incr_20260424 /my_dir/audit_incr_20260424
```

For StorageClass deployments, replace `/cpaas/data/clickhouse/backups` with the actual ClickHouse backup directory on the StorageClass volume.

Notes:

- `/my_dir` can be a designated local archive directory or a NAS mount path.
- Use an archive path outside the ClickHouse data directory. Otherwise, the backup files might be deleted when the ClickHouse data directory is cleaned during disaster recovery.
- After each full or incremental backup, copy the corresponding backup directory to the archive location.
- After the incremental backup is copied successfully, you can delete the local incremental backup under the ClickHouse backup directory if it is no longer needed.
- Delete the local full backup under the ClickHouse backup directory only after the next full backup succeeds and the retention policy allows cleanup.
- If you later restore with `RESTORE ... ON CLUSTER ... FROM File(...)`, make sure the required full and incremental backup directories are copied back to the ClickHouse backup directory on each ClickHouse host node before running the restore command.

## Restore

Use this procedure when table data is corrupted, the data directory is deleted, or the table state is abnormal.

### Restore Prerequisites

Before you start the restore procedure, make sure the following conditions are met:

- You have a valid full backup or incremental backup.
- If you restore from an incremental backup, the corresponding full backup is available in the restore path.
- You know the archive directory that stores the latest incremental backup and the corresponding full backup.
- You have access to the Kubernetes cluster and the host nodes where ClickHouse instances are scheduled.

### Restore Procedure

This procedure restores ClickHouse data table by table. Choose the preparation steps according to the failure scope, and then run the same table restore procedure for each required table.

#### Stop Writes

Stop `razor` first to prevent new data from being written during the restore.

Log in to the cluster master node and create a ResourcePatch to stop `razor`:

```bash
cat <<EOF > /tmp/rp-stop-razor.yaml
apiVersion: operator.alauda.io/v1alpha1
kind: ResourcePatch
metadata:
  generateName: rp-
  name: rp-stop-razor
spec:
  jsonPatch:
  - op: replace
    path: /spec/replicas
    value: 0
  release: cpaas-system/logclickhouse
  target:
    apiVersion: apps/v1
    kind: StatefulSet
    name: razor
    namespace: cpaas-system
EOF

kubectl apply -f /tmp/rp-stop-razor.yaml
```

#### Prepare ClickHouse According to the Failure Scope

Choose one of the following preparation paths according to the failure scope. After the preparation is complete, continue with the same table-by-table restore procedure.

##### Case A: Table Data Is Corrupted but the Data Directory Is Healthy

Use this case when one or more tables are corrupted, accidentally deleted, or have abnormal data, while the ClickHouse data directory and Keeper state are still healthy.

In this case, do not stop ClickHouse and do not clean `/cpaas/data/clickhouse/*`. Continue with the table restore procedure directly.

##### Case B: Data Directory or Keeper Metadata Is Damaged

Use this case only when the ClickHouse data directory is damaged, the node is rebuilt, or the Keeper metadata is unavailable.

In this deployment, ClickHouse Keeper is integrated with ClickHouse and its data is also stored under the ClickHouse data directory. Therefore, cleaning `/cpaas/data/clickhouse/*` also removes the local ClickHouse Keeper data.

> Warning:
> Cleaning the data directory deletes local ClickHouse data on the target nodes. Confirm that the backup is available before you continue.
>
> Run the `rm -rf /cpaas/data/clickhouse/*` command on each ClickHouse host node, not inside the ClickHouse container.

Log in to the cluster master node and create a ResourcePatch to stop ClickHouse:

```bash
cat <<EOF > /tmp/rp-stop-ck.yaml
apiVersion: operator.alauda.io/v1alpha1
kind: ResourcePatch
metadata:
  generateName: rp-
  name: rp-stop-ck
spec:
  jsonPatch:
  - op: add
    path: /spec/stop
    value: "true"
  release: cpaas-system/logclickhouse
  target:
    apiVersion: clickhouse.altinity.com/v1
    kind: ClickHouseInstallation
    name: cpaas-clickhouse
    namespace: cpaas-system
EOF

kubectl apply -f /tmp/rp-stop-ck.yaml
```

Confirm that all ClickHouse Pods have stopped:

```bash
kubectl get pod -n cpaas-system | grep -i "chi-cpaas-clickhouse-replicated"
```

On each host node where a ClickHouse instance is deployed, clean the local ClickHouse data directory:

```bash
rm -rf /cpaas/data/clickhouse/*
```

Start only the ClickHouse components by deleting the ClickHouse ResourcePatch. Do not start `razor` yet.

```bash
kubectl delete -f /tmp/rp-stop-ck.yaml
```

Confirm that all ClickHouse Pods are running:

```bash
kubectl get pod -n cpaas-system | grep -i "chi-cpaas-clickhouse-replicated"
```

#### Prepare Backup Files on Each ClickHouse Node

Make sure the backup files are available in the ClickHouse backup directory on each ClickHouse host node.

If the backup files were archived to a custom backup directory or NAS mount path and the local files under the ClickHouse backup directory have been deleted, copy the latest incremental backup and the corresponding full backup back to the ClickHouse backup directory on each ClickHouse host node before you restore the table.

For LocalVolume deployments, the destination path is typically `/cpaas/data/clickhouse/backups`:

```bash
mkdir -p /cpaas/data/clickhouse/backups
cp -r /my_dir/audit_full_20260423 /cpaas/data/clickhouse/backups/audit_full_20260423
cp -r /my_dir/audit_incr_20260424 /cpaas/data/clickhouse/backups/audit_incr_20260424
```

For StorageClass deployments, copy the backup files back to the actual ClickHouse backup directory on the StorageClass volume.

Notes:

- If you restore from an incremental backup, prepare the dependent full backup at the same time.
- The backup files must be placed under the ClickHouse backup directory on each ClickHouse host node so that `RESTORE ... ON CLUSTER ... FROM File(...)` can access them on every ClickHouse instance.
- Replace `/my_dir`, `/cpaas/data/clickhouse/backups`, `audit_full_20260423`, and `audit_incr_20260424` with the actual archive path, ClickHouse backup directory, and backup directory names.

#### Check Cluster Readiness

Before you run the restore command, make sure the ClickHouse cluster configuration and macros are available.

Check the `replicated` cluster configuration:

```sql
SELECT *
FROM system.clusters
WHERE cluster = 'replicated';
```

Check the local ClickHouse macros:

```sql
SELECT *
FROM system.macros;
```

Make sure the cluster contains the expected ClickHouse replicas and the macros such as `shard` and `replica` are available.

#### Restore Tables One by One

Run the following restore procedure for each table that needs to be restored.

Drop the target table on the ClickHouse cluster:

```sql
DROP TABLE observability.audit ON CLUSTER 'replicated' SYNC;
```

Restore the table from the local or NAS backup:

```sql
RESTORE TABLE observability.audit
ON CLUSTER 'replicated'
FROM File('audit_incr_20260424');
```

Notes:

- For a `ReplicatedMergeTree` table in this 3-replica deployment, use `ON CLUSTER 'replicated'` so that the restore operation is distributed to all ClickHouse replicas in the cluster.
- If you restore from an incremental backup, ClickHouse reads the dependent base backup automatically.
- Keep the corresponding full backup directory accessible during the restore.
- Replace `observability.audit` and the backup directory name with the actual table and backup that you want to restore.
- Repeat the same procedure for every table that needs to be restored. The table list is determined by the customer.

#### Check Restore Task Status

After each restore command is executed, check the restore task status before you start any related components:

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

The restore is successful if the following conditions are met:

- `status` indicates that the restore operation has completed successfully.
- `error` is empty.
- `end_time` has a value.

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

## Start the Related Components

After the restore has been validated successfully, start `razor` again by deleting the ResourcePatch:

```bash
kubectl delete -f /tmp/rp-stop-razor.yaml
```

## Recommendations

Use the following recommendations in production:

- Maintain the backup chain with one full backup per week and one incremental backup per day.
- Use a consistent naming convention for backup directories, such as `full_YYYYMMDD` and `incr_YYYYMMDD`.
- Run restore drills in a test environment on a regular basis.
- Define a retention policy for expired incremental backups and historical full backups.
