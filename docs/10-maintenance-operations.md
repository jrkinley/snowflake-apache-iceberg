# Maintenance Operations & Table Health

## Why Maintenance Matters

Iceberg tables accumulate metadata and small files over time. Regular maintenance ensures:
- Optimal query performance
- Controlled storage costs
- Efficient metadata operations

## Common Maintenance Tasks

| Task | Purpose | Frequency |
|------|---------|-----------|
| **Compaction** | Merge small files | Daily/Weekly |
| **Snapshot Expiration** | Remove old snapshots | Weekly |
| **Manifest Rewrite** | Optimize manifest files | Monthly |
| **Orphan File Cleanup** | Remove unreferenced files | Monthly |
| **Statistics Update** | Refresh column stats | After major changes |

## Compaction

### The Small Files Problem

```
Before Compaction:          After Compaction:
├── file-001.parquet (5MB)  ├── file-001.parquet (150MB)
├── file-002.parquet (3MB)  ├── file-002.parquet (150MB)
├── file-003.parquet (8MB)  └── file-003.parquet (120MB)
├── file-004.parquet (2MB)
├── ... (100 more files)
```

### Compaction in Snowflake

```sql
-- Compact all data files
ALTER ICEBERG TABLE my_table COMPACT DATA;

-- Compact with target file size
ALTER ICEBERG TABLE my_table COMPACT DATA
  TARGET_FILE_SIZE_BYTES = 134217728;  -- 128MB
```

### When to Compact

- After many small streaming inserts
- Query performance degrades
- File count exceeds recommended thresholds
- Before major analytical workloads

## Snapshot Management

### Expire Old Snapshots

```sql
-- Expire snapshots older than timestamp
ALTER ICEBERG TABLE my_table 
  EXPIRE SNAPSHOTS OLDER_THAN TIMESTAMP '2024-01-01 00:00:00';

-- Keep only last N snapshots
ALTER ICEBERG TABLE my_table 
  EXPIRE SNAPSHOTS RETAIN_LAST 5;
```

### What Expiration Removes

| Removed | Kept |
|---------|------|
| Old snapshot entries | Current snapshot |
| Unreferenced manifest lists | Referenced manifests |
| Unreferenced manifest files | Active data files |
| Orphaned data files | Files in current snapshot |

### Considerations

- Cannot time travel to expired snapshots
- Irreversible operation
- Run during low-activity periods

## Manifest Optimization

### Rewrite Manifests

Consolidate small manifests for faster planning:

```sql
-- Rewrite manifests to optimal size
ALTER ICEBERG TABLE my_table REWRITE MANIFESTS;
```

### When to Rewrite

- Many small manifests (>100)
- Query planning is slow
- After many incremental writes

## Orphan File Cleanup

Files not referenced by any snapshot:

```sql
-- Remove orphan files (use with caution)
-- Ensure no active queries before running
ALTER ICEBERG TABLE my_table 
  REMOVE ORPHAN FILES OLDER_THAN TIMESTAMP '2024-01-01';
```

**Causes of Orphan Files:**
- Failed write operations
- Concurrent write conflicts
- Incomplete transactions

## Table Health Monitoring

### Key Metrics to Track

| Metric | Healthy Range | Action if Exceeded |
|--------|--------------|-------------------|
| File count | < 10,000 per partition | Compact |
| Avg file size | 100MB - 1GB | Compact or adjust writes |
| Snapshot count | < 100 | Expire old snapshots |
| Manifest count | < 100 | Rewrite manifests |
| Delete file ratio | < 10% of data files | Compact |

### Monitoring Queries

```sql
-- Table metadata overview
DESCRIBE ICEBERG TABLE my_table;

-- Check table properties
SHOW ICEBERG TABLES LIKE 'my_table';

-- File statistics (approximate)
SELECT 
  COUNT(*) as file_count,
  SUM(file_size_in_bytes) as total_bytes,
  AVG(file_size_in_bytes) as avg_file_size
FROM TABLE(my_table$FILES);
```

## Maintenance Schedule Template

### Daily

```sql
-- Lightweight operations
-- Monitor for small file accumulation
```

### Weekly

```sql
-- Compact if needed
ALTER ICEBERG TABLE my_table COMPACT DATA;

-- Expire old snapshots
ALTER ICEBERG TABLE my_table 
  EXPIRE SNAPSHOTS RETAIN_LAST 10;
```

### Monthly

```sql
-- Full maintenance
ALTER ICEBERG TABLE my_table COMPACT DATA;
ALTER ICEBERG TABLE my_table REWRITE MANIFESTS;
ALTER ICEBERG TABLE my_table 
  EXPIRE SNAPSHOTS OLDER_THAN CURRENT_TIMESTAMP - INTERVAL '30 DAYS';
```

## Automation with Tasks

```sql
-- Create scheduled maintenance task
CREATE TASK iceberg_maintenance_task
  WAREHOUSE = maintenance_wh
  SCHEDULE = 'USING CRON 0 2 * * 0 America/Los_Angeles'  -- Weekly Sunday 2 AM
AS
  ALTER ICEBERG TABLE my_table COMPACT DATA;

-- Enable the task
ALTER TASK iceberg_maintenance_task RESUME;
```

## Maintenance for External Catalog Tables

For tables using external catalogs (Glue, REST):

- Maintenance operations depend on catalog capabilities
- Some operations may need external tools (Spark, etc.)
- Coordinate with external catalog's maintenance procedures

## Best Practices

1. **Automate maintenance**: Use Snowflake Tasks for scheduling
2. **Monitor before acting**: Check metrics before aggressive cleanup
3. **Maintenance windows**: Run during low-activity periods
4. **Test in non-prod**: Verify maintenance impact before production
5. **Document retention policies**: Align snapshot expiration with compliance
6. **Balance frequency vs. cost**: Maintenance consumes compute resources
