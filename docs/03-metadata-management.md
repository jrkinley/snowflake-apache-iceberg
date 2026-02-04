# Metadata Management

## Metadata Layer Overview

The metadata layer is the **core innovation** of Apache Iceberg. It provides file-level tracking of every data file without requiring directory listing operations — critical for cloud object storage performance.

### The Directory Listing Problem

Cloud object storage is designed for high-throughput reads/writes, not filesystem operations:
- **LIST operations are expensive**: Each call returns limited results (typically 1,000 objects), requiring pagination for large directories
- **High latency**: LIST requests have higher latency than GET requests — scanning thousands of files can take seconds to minutes
- **Cost**: Cloud providers charge per API call; listing large tables repeatedly adds up
- **No atomic directory operations**: Unlike filesystems, object storage has no concept of directories — they're simulated via key prefixes

### How File-Level Tracking Solves This

Iceberg's manifest files explicitly enumerate every data file with its metadata. Query engines read a small number of manifest files (few MBs) instead of listing potentially millions of objects. This is why Iceberg can scale to tables with millions of files.

## Metadata Components

```
┌─────────────────────────────────────────────────────────┐
│                  METADATA FILE (JSON)                   │
│  • Current schema                                       │
│  • Partition spec                                       │
│  • Snapshot list & current snapshot ID                  │
│  • Table properties                                     │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                 MANIFEST LIST (Avro)                    │
│  • Points to manifest files                             │
│  • Partition summary statistics                         │
│  • Tracks added/deleted manifests per snapshot          │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                  MANIFEST FILE (Avro)                   │
│  • List of data file paths                              │
│  • Per-file column statistics (min/max/nulls)           │
│  • Partition values for each file                       │
│  • File status (added/existing/deleted)                 │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                   DATA FILES (Parquet)                  │
└─────────────────────────────────────────────────────────┘
```

## Metadata File Structure

```
{
  "format-version": 3,                    // Iceberg spec version (1, 2, or 3)
  "table-uuid": "abc123...",
  "location": "s3://bucket/table",        // Base path for table data and metadata
  "last-sequence-number": 5,              // Monotonic counter for ordering changes
  "last-updated-ms": 1706000000000,
  "current-snapshot-id": 123456789,       // Pointer to the active snapshot
  "schemas": [...],                       // All schema versions; each snapshot references its schema ID
  "current-schema-id": 0,                 // Active schema version ID
  "partition-specs": [...],               // Partition rules (columns + transforms like day(), bucket())
  "default-spec-id": 0,                   // Active partition spec ID
  "snapshots": [
    {
      "snapshot-id": 123456789,
      "timestamp-ms": 1706000000000,
      "manifest-list": "s3://...snap.avro" // Path to this snapshot's manifest list
    }
  ]
}
```

## Manifest List Contents

Each snapshot points to a manifest list (Avro file) that indexes all manifest files for that snapshot:

| Field | Description |
|-------|-------------|
| `manifest_path` | Location of the manifest file |
| `manifest_length` | Size of manifest file in bytes |
| `partition_spec_id` | Partition spec used by this manifest |
| `added_snapshot_id` | Snapshot that added this manifest |
| `added_files_count` | Number of data files added |
| `existing_files_count` | Number of existing data files |
| `deleted_files_count` | Number of data files marked for deletion |
| `partitions` | Summary of partition values in this manifest |

> **Note:** The manifest list enables efficient snapshot-level operations — Iceberg can determine which manifests changed between snapshots without reading all manifest files.

## Manifest File Contents

Each manifest file tracks multiple data files:

| Field | Description |
|-------|-------------|
| `file_path` | Location of data file |
| `file_format` | PARQUET (Snowflake), ORC, or AVRO |
| `partition` | Partition tuple values |
| `record_count` | Number of rows |
| `file_size_in_bytes` | File size |
| `column_sizes` | Size per column ID |
| `value_counts` | Non-null count per column |
| `null_value_counts` | Null count per column |
| `lower_bounds` | Min value per column |
| `upper_bounds` | Max value per column |

## Statistics for Query Planning

### Column-Level Statistics

Column-level statistics (min/max values, null counts) stored in manifest files enable **predicate pushdown** — filtering happens at the metadata layer before reading any data files. This dramatically reduces I/O by skipping files that cannot contain matching rows.

```
Query: SELECT * FROM otel_traces
       WHERE DATE(start_time) = '2026-02-04'
         AND service_name = 'payment-service'

Manifest Entry:
  file: traces/date=2026-02-04/service=payment-service/00001.parquet
  lower_bounds: {start_time: '2026-02-04 00:00:00', service_name: 'payment-service'}
  upper_bounds: {start_time: '2026-02-04 23:59:59', service_name: 'payment-service'}
  → FILE INCLUDED (partition match)

Manifest Entry:
  file: traces/date=2026-02-04/service=auth-service/00002.parquet
  lower_bounds: {start_time: '2026-02-04 00:00:00', service_name: 'auth-service'}
  upper_bounds: {start_time: '2026-02-04 23:59:59', service_name: 'auth-service'}
  → FILE SKIPPED (service_name mismatch)
```

### Partition Statistics

Manifest lists contain partition summaries for fast partition pruning:

```
Manifest: traces/metadata/manifest-001.avro
  partition_summaries:
    - field: DATE(start_time)
      contains_null: false
      lower_bound: 2026-02-01
      upper_bound: 2026-02-04
    - field: service_name
      contains_null: false
      lower_bound: 'api-gateway'
      upper_bound: 'payment-service'
```

## Metadata Growth & Cleanup

| Operation | Metadata Impact |
|-----------|-----------------|
| INSERT | New manifest file(s) added |
| DELETE | Files marked as deleted in manifest |
| UPDATE | Delete + Insert (copy-on-write) |
| COMPACT | Rewrite manifests, expire old snapshots |

### Inspecting Iceberg Tables

```sql
-- View table metadata location and properties
DESCRIBE ICEBERG TABLE <table>;

-- Show table structure and column types
DESCRIBE TABLE <table>;

-- View current snapshot and metadata file location
SELECT SYSTEM$GET_ICEBERG_TABLE_INFORMATION('<table>');

-- List all snapshots with timestamps
SELECT * FROM TABLE(INFORMATION_SCHEMA.ICEBERG_TABLE_SNAPSHOTS('<table>'));

-- View manifest files for current snapshot
SELECT * FROM TABLE(INFORMATION_SCHEMA.ICEBERG_TABLE_MANIFESTS('<table>'));

-- Check table storage metrics
SELECT * FROM TABLE(INFORMATION_SCHEMA.ICEBERG_TABLE_FILES('<table>'));
```

### Compaction & Optimization

```sql
-- Compact small files into larger optimized files
ALTER ICEBERG TABLE <table> COMPACT;

-- Rewrite manifests for better read performance (reduces manifest count)
ALTER ICEBERG TABLE <table> REWRITE MANIFESTS;

-- Optimize specific partitions
ALTER ICEBERG TABLE <table> COMPACT
  WHERE <partition_column> = <value>;
```

### Snapshot Cleanup

```sql
-- Expire snapshots older than a specific timestamp
ALTER ICEBERG TABLE <table>
  EXPIRE SNAPSHOTS OLDER_THAN TIMESTAMP '<timestamp>';

-- Expire snapshots keeping only the last N snapshots
ALTER ICEBERG TABLE <table>
  EXPIRE SNAPSHOTS RETAIN LAST <n>;

-- Remove orphaned files not referenced by any snapshot
ALTER ICEBERG TABLE <table> REMOVE ORPHAN FILES;
```

> **Note:** For Snowflake-managed Iceberg tables (`CATALOG = 'SNOWFLAKE'`), automatic cleanup runs in the background based on `DATA_RETENTION_TIME_IN_DAYS`. Manual cleanup commands are useful for immediate space reclamation or external catalog tables.

## Best Practices

**Apache Iceberg recommendations:**
- **Expire snapshots regularly** — unbounded snapshot growth increases metadata size and slows query planning
- **Compact manifests** — many small manifests from streaming ingestion degrade read performance
- **Remove orphan files** — failed writes can leave unreferenced data files consuming storage
- **Limit partition columns** — each partition spec version is retained in metadata; excessive partitioning increases complexity

**Snowflake recommendations:**
- **Set `DATA_RETENTION_TIME_IN_DAYS` appropriately** — balance time travel needs against storage costs (default: 1 day, max: 90 days)
- **Use Snowflake-managed tables when possible** — automatic maintenance handles compaction, cleanup, and optimization
- **Monitor with `SYSTEM$GET_ICEBERG_TABLE_INFORMATION`** — track metadata file locations and snapshot health
- **Leverage automatic clustering** — unlike compaction (which merges small files into larger ones), clustering re-organizes data across files to co-locate similar values by cluster key, improving file pruning effectiveness on frequently filtered columns
