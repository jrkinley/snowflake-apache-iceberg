# Metadata Management

## Metadata Layer Overview

The metadata layer is the **core innovation** of Iceberg. It provides file-level tracking without requiring directory listing operations—critical for cloud object storage performance.

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
│  • Per-file column statistics (min/max/nulls)          │
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

```json
{
  "format-version": 2,
  "table-uuid": "abc123...",
  "location": "s3://bucket/table",
  "last-sequence-number": 5,
  "last-updated-ms": 1706000000000,
  "current-snapshot-id": 123456789,
  "schemas": [...],
  "current-schema-id": 0,
  "partition-specs": [...],
  "default-spec-id": 0,
  "snapshots": [
    {
      "snapshot-id": 123456789,
      "timestamp-ms": 1706000000000,
      "manifest-list": "s3://bucket/table/metadata/snap-123.avro"
    }
  ]
}
```

## Manifest File Contents

Each manifest file tracks multiple data files:

| Field | Description |
|-------|-------------|
| `file_path` | Location of data file |
| `file_format` | PARQUET, ORC, or AVRO |
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

```
Query: SELECT * FROM orders WHERE order_date = '2024-01-15'

Manifest Entry:
  file: data/00001.parquet
  lower_bounds: {order_date: '2024-01-01'}
  upper_bounds: {order_date: '2024-01-31'}
  → FILE INCLUDED (date in range)

Manifest Entry:
  file: data/00002.parquet
  lower_bounds: {order_date: '2024-02-01'}
  upper_bounds: {order_date: '2024-02-28'}
  → FILE SKIPPED (date outside range)
```

### Partition Statistics

Manifest lists contain partition summaries for fast partition pruning:

```
Manifest: manifest-001.avro
  partition_summaries:
    - contains_null: false
      lower_bound: 2024-01-01
      upper_bound: 2024-01-31
```

## Metadata Operations

### Commit Process

1. Write new data files
2. Create new manifest file(s)
3. Create new manifest list
4. Write new metadata file
5. Atomically update catalog pointer

### Optimistic Concurrency

```
Writer A: Read metadata v1
Writer B: Read metadata v1
Writer A: Write metadata v2 ← SUCCESS
Writer B: Write metadata v2 ← CONFLICT (retry from v2)
```

## Snowflake Metadata Handling

When `CATALOG = 'SNOWFLAKE'`:

- Snowflake manages all metadata files automatically
- Metadata stored in `BASE_LOCATION/metadata/`
- Query optimizer leverages manifest statistics
- No manual metadata maintenance required

```sql
-- View metadata file location
DESCRIBE ICEBERG TABLE my_table;
```

## Metadata Growth & Cleanup

| Operation | Metadata Impact |
|-----------|-----------------|
| INSERT | New manifest file(s) added |
| DELETE | Files marked as deleted in manifest |
| UPDATE | Delete + Insert (copy-on-write) |
| COMPACT | Rewrite manifests, expire old snapshots |

### Cleanup Commands

```sql
-- Remove old snapshots (Snowflake-managed tables)
ALTER ICEBERG TABLE my_table EXPIRE SNAPSHOTS OLDER_THAN TIMESTAMP '2024-01-01';

-- Rewrite manifests for better read performance
ALTER ICEBERG TABLE my_table REWRITE MANIFESTS;
```

## Best Practices

1. **Monitor manifest count**: Many small manifests slow planning
2. **Regular compaction**: Combine small manifests periodically
3. **Expire old snapshots**: Reduce metadata file count
4. **Use appropriate statistics**: Column stats enable pruning
