# ACID Transactions & Concurrency Control

## ACID Properties in Iceberg

Iceberg provides full **ACID guarantees** for table operations:

| Property | Iceberg Implementation |
|----------|----------------------|
| **Atomicity** | All changes in a commit succeed or none do |
| **Consistency** | Table always in valid state between commits |
| **Isolation** | Concurrent readers see consistent snapshots |
| **Durability** | Committed changes persisted to storage |

## How Iceberg Achieves ACID

### Atomic Commits via Metadata Pointer

```
Before:  catalog → metadata-v1.json → snapshot-100
After:   catalog → metadata-v2.json → snapshot-101
                          │
                          └── Atomic pointer swap
```

1. Write data files to object storage
2. Create new metadata file (references new + existing files)
3. Atomically update catalog pointer to new metadata
4. Old metadata/files remain for time travel

### Snapshot Isolation

Every query sees a **consistent snapshot**:

```
Timeline:
─────────────────────────────────────────────────────►

Writer:      [Write S1] ─────────── [Commit S2]
                  │                      │
Query A:         [Start]────────────────[End] → Sees S1
                           │
Query B:                  [Start]─────────────[End] → Sees S1
                                         │
Query C:                                [Start]────[End] → Sees S2
```

- Long-running queries unaffected by concurrent writes
- No dirty reads, no phantom reads
- Each query bound to single snapshot at start

## Concurrency Control

### Optimistic Concurrency

Iceberg uses **optimistic locking** for writes:

```
Writer A: Read metadata v1
Writer B: Read metadata v1

Writer A: Create new files, write metadata v2
Writer A: CAS(v1 → v2) ✓ SUCCESS

Writer B: Create new files, write metadata v2'
Writer B: CAS(v1 → v2') ✗ CONFLICT (v1 changed)
Writer B: Retry from v2
```

**CAS = Compare-And-Swap**: Atomic operation ensuring no lost updates

### Conflict Resolution

| Scenario | Resolution |
|----------|------------|
| Both append to different partitions | Auto-retry succeeds |
| Both append to same partition | Auto-retry succeeds |
| Overlapping deletes | Retry with merged view |
| Conflicting schema changes | Retry may fail |

## Write Operations in Snowflake

### Insert (Append)

```sql
INSERT INTO my_iceberg_table (id, name, value)
VALUES (1, 'Alice', 100);

-- Bulk insert
INSERT INTO my_iceberg_table
SELECT * FROM staging_table;
```

- Creates new data files
- New snapshot references old + new files
- Highly concurrent (appends rarely conflict)

### Delete

```sql
DELETE FROM my_iceberg_table
WHERE id = 100;
```

**Copy-on-Write (default):**
- Rewrites affected files without deleted rows
- New snapshot points to rewritten files

**Merge-on-Read (optional):**
- Writes delete file marking rows as deleted
- Reads merge data + delete files

```sql
-- Enable merge-on-read for table
ALTER ICEBERG TABLE my_table 
  SET ENABLE_ICEBERG_MERGE_ON_READ = TRUE;
```

### Update

```sql
UPDATE my_iceberg_table
SET value = value + 10
WHERE status = 'pending';
```

Implemented as DELETE + INSERT:
1. Identify affected rows
2. Write new files with updated values
3. Mark old rows as deleted

### Merge

```sql
MERGE INTO target t
USING source s ON t.id = s.id
WHEN MATCHED AND s.deleted THEN DELETE
WHEN MATCHED THEN UPDATE SET t.value = s.value
WHEN NOT MATCHED THEN INSERT (id, value) VALUES (s.id, s.value);
```

Combines conditional INSERT, UPDATE, DELETE in single transaction.

## Transaction Limitations

### Snowflake-Managed Tables

- Full transaction support within single statement
- Multi-statement transactions supported via Snowflake

### Externally Managed Tables

- **Autocommit only**: Each statement is its own transaction
- Multi-statement transactions NOT supported
- Each DML = one Iceberg commit

```sql
-- This works (single statement)
INSERT INTO ext_iceberg_table SELECT * FROM source;

-- Multi-statement transactions NOT supported for external catalogs
BEGIN;
  INSERT INTO ext_iceberg_table VALUES (1);
  INSERT INTO ext_iceberg_table VALUES (2);
COMMIT;  -- ❌ Not atomic for external Iceberg tables
```

## Row-Level Operations

### Position Deletes

Iceberg v2 supports tracking deleted rows by file + position:

```
Delete File:
  file_path: data/00001.parquet
  positions: [10, 25, 99]  -- Row positions to skip
```

### Equality Deletes

Delete by column values (not currently supported in Snowflake):

```
Delete File:
  equality_columns: [id]
  values: [100, 200, 300]  -- Rows with these IDs deleted
```

## Best Practices

1. **Batch writes**: Combine small writes to reduce snapshots
2. **Avoid row-level updates on large datasets**: Prefer batch operations
3. **Use MERGE for upserts**: More efficient than DELETE + INSERT
4. **Monitor for conflicts**: High contention → consider partitioning
5. **Compact regularly**: Merge-on-read accumulates delete files
6. **Understand isolation**: Queries see start-time snapshot
