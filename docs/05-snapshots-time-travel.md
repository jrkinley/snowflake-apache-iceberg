# Snapshot Management & Time Travel

## What is a Snapshot?

A snapshot represents the **complete state of a table at a point in time**. It's a pointer to a manifest list that tracks all data files belonging to that version.

```
Snapshot 1 ──► Manifest List ──► [file1, file2, file3]
     │
     ▼ (INSERT)
Snapshot 2 ──► Manifest List ──► [file1, file2, file3, file4]
     │
     ▼ (DELETE)
Snapshot 3 ──► Manifest List ──► [file1, file2, file4]
```

## Snapshot Properties

| Property | Description |
|----------|-------------|
| `snapshot-id` | Unique identifier (64-bit long) |
| `timestamp-ms` | Creation timestamp |
| `manifest-list` | Path to manifest list file |
| `summary` | Operation type, added/deleted files count |
| `parent-snapshot-id` | Previous snapshot in lineage |

## Snapshot Operations

### Write Operations Create Snapshots

| Operation | Snapshot Action |
|-----------|-----------------|
| INSERT | Add new data files |
| DELETE | Mark files as deleted |
| UPDATE | Delete + Insert (copy-on-write by default) |
| MERGE | Combination based on matched/unmatched |
| COMPACT | Rewrite files, single new snapshot |

### Snapshot Metadata Example

```json
{
  "snapshot-id": 3497810834902348,
  "timestamp-ms": 1706000000000,
  "summary": {
    "operation": "append",
    "added-data-files": "5",
    "added-records": "100000",
    "total-records": "500000",
    "total-data-files": "25"
  },
  "manifest-list": "s3://bucket/table/metadata/snap-3497810834902348.avro"
}
```

## Time Travel in Snowflake

### Query Historical Snapshots

```sql
-- By timestamp
SELECT * FROM my_iceberg_table
  AT (TIMESTAMP => '2024-01-15 10:00:00'::TIMESTAMP);

-- By offset (relative time)
SELECT * FROM my_iceberg_table
  AT (OFFSET => -3600);  -- 1 hour ago

-- By snapshot ID (Snowflake-managed tables)
SELECT * FROM my_iceberg_table
  BEFORE (STATEMENT => '<query_id>');
```

### View Snapshot History

```sql
-- List snapshots for Snowflake-managed Iceberg table
SELECT * 
FROM TABLE(INFORMATION_SCHEMA.ICEBERG_TABLE_SNAPSHOTS('my_iceberg_table'));
```

## Snapshot Retention

### Snowflake-Managed Tables

Retention controlled by table parameter:

```sql
-- Set retention period
ALTER ICEBERG TABLE my_table 
  SET DATA_RETENTION_TIME_IN_DAYS = 7;

-- Check current setting
SHOW PARAMETERS LIKE 'DATA_RETENTION%' IN TABLE my_table;
```

### External Catalog Tables

Retention managed by external catalog/tools. Snowflake reads whatever snapshots exist.

## Snapshot Expiration

Remove old snapshots to reduce metadata size and storage costs:

```sql
-- Expire snapshots older than timestamp
ALTER ICEBERG TABLE my_table 
  EXPIRE SNAPSHOTS OLDER_THAN TIMESTAMP '2024-01-01 00:00:00';

-- Keep only last N snapshots
ALTER ICEBERG TABLE my_table 
  EXPIRE SNAPSHOTS RETAIN_LAST 10;
```

**What happens during expiration:**
1. Old snapshot entries removed from metadata
2. Manifest files no longer referenced are deleted
3. Data files no longer referenced by any snapshot are deleted
4. Current snapshot and ancestors within retention are preserved

## Snapshot Isolation

Iceberg provides **serializable isolation** for readers:

```
Time ──────────────────────────────────────────►

Writer:  [Start TX] ─────── [Commit Snap 2]
              │                    │
Reader A:    [Start Query @ Snap 1] ─── [Returns Snap 1 data]
              │
Reader B:         [Start Query @ Snap 1] ─── [Returns Snap 1 data]
                                        │
Reader C:                              [Start Query @ Snap 2] ─── [Returns Snap 2]
```

- Readers always see consistent snapshot
- Long-running queries unaffected by concurrent writes
- No dirty reads or phantom reads

## Branching & Tagging (Iceberg v2)

### Tags (Named Snapshots)

```sql
-- Create a tag for a specific snapshot (via Spark/external tools)
-- Useful for marking releases or checkpoints
```

### Branches (Parallel Lineages)

```
main:     S1 ─► S2 ─► S3 ─► S4
                 │
                 └─► S2a ─► S2b  :audit-branch
```

*Note: Branch/tag support in Snowflake depends on catalog capabilities*

## Cherry-Pick & Rollback

### Rollback to Previous Snapshot

```sql
-- Rollback to specific snapshot (creates new snapshot pointing to old state)
-- Implementation varies by catalog
```

### Fast-Forward (Cherry-Pick)

Apply changes from one branch to another without full merge.

## Best Practices

1. **Set appropriate retention**: Balance time travel needs vs. storage costs
2. **Regular snapshot expiration**: Prevent unbounded metadata growth
3. **Monitor snapshot count**: Too many snapshots slow metadata operations
4. **Use tags for important states**: Mark releases, audits, backups
5. **Understand isolation guarantees**: Queries see consistent point-in-time view
