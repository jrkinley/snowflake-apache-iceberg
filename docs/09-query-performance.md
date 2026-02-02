# Query Planning & Performance Optimization

## How Iceberg Optimizes Queries

Iceberg's metadata structure enables efficient query planning **without listing directories**:

```
Traditional Data Lake:          Iceberg:
                               
LIST bucket/table/             READ manifest list (1 file)
  └─► 1000s of requests        READ manifests (few files)
  └─► Slow, expensive          └─► Contains all file info
                               └─► Fast, predictable
```

## Query Planning Phases

### Phase 1: Partition Pruning

Filter partitions using manifest list summaries:

```
Query: WHERE date = '2024-01-15' AND region = 'US'

Manifest List contains:
  manifest-001.avro: date range [2024-01-01, 2024-01-31], region: US ✓
  manifest-002.avro: date range [2024-02-01, 2024-02-28], region: US ✗
  manifest-003.avro: date range [2024-01-01, 2024-01-31], region: EU ✗
```

Result: Only read `manifest-001.avro`

### Phase 2: File Pruning (Data Skipping)

Filter files using column statistics in manifests:

```
Manifest-001 contains files:
  file-001.parquet: date [2024-01-01, 2024-01-10] ✗
  file-002.parquet: date [2024-01-11, 2024-01-20] ✓ Contains 2024-01-15
  file-003.parquet: date [2024-01-21, 2024-01-31] ✗
```

Result: Only scan `file-002.parquet`

### Phase 3: Row Group Pruning (Parquet)

Within files, use Parquet statistics to skip row groups:

```
file-002.parquet:
  Row Group 1: date [2024-01-11, 2024-01-13] ✗
  Row Group 2: date [2024-01-14, 2024-01-16] ✓
  Row Group 3: date [2024-01-17, 2024-01-20] ✗
```

## Statistics for Pruning

### Column-Level Statistics

Stored in manifest files:

| Statistic | Use |
|-----------|-----|
| `lower_bounds` | Min value per column |
| `upper_bounds` | Max value per column |
| `null_value_counts` | Nulls per column |
| `value_counts` | Non-null count |
| `column_sizes` | Bytes per column |

### Enabling Statistics

```sql
-- Snowflake collects statistics automatically for managed tables
-- For optimal pruning, filter on columns with good min/max distribution
```

## Snowflake Query Optimization

### Automatic Optimizations

| Optimization | Description |
|--------------|-------------|
| **Predicate pushdown** | Filters applied during file scan |
| **Column pruning** | Only read required columns |
| **Partition pruning** | Skip non-matching partitions |
| **File pruning** | Skip files based on statistics |
| **Parallel execution** | Scan files concurrently |

### Query Profile Analysis

```sql
-- Check query performance
SELECT * FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
WHERE QUERY_ID = '<your_query_id>';

-- View files scanned
SELECT * FROM TABLE(RESULT_SCAN('<your_query_id>'));
```

## Performance Patterns

### Good: Partition-Aligned Filters

```sql
-- Table partitioned by day(event_time)
-- Filter aligns with partition → excellent pruning
SELECT * FROM events
WHERE event_time >= '2024-01-15' AND event_time < '2024-01-16';
```

### Problematic: Non-Selective Filters

```sql
-- Filter doesn't match partition or has poor selectivity
SELECT * FROM events
WHERE event_type LIKE '%error%';  -- Full scan, no pruning
```

### Optimal: Combine Partition + Data Filters

```sql
SELECT * FROM events
WHERE event_time >= '2024-01-15'     -- Partition prune
  AND event_time < '2024-01-16'
  AND user_id = 12345;               -- Data file prune via stats
```

## File Organization Impact

### Small Files Problem

```
1000 files × 1MB = Poor performance
- High manifest overhead
- Many file opens
- Reduced parallelism efficiency
```

### Ideal File Sizes

```
10 files × 100MB = Good performance
- Efficient manifest
- Optimal I/O
- Good parallelism
```

### Compaction

```sql
-- Compact small files (Snowflake)
ALTER ICEBERG TABLE my_table COMPACT DATA;
```

## Clustering (Data Organization)

For better data skipping within partitions:

```sql
-- Cluster data by frequently filtered columns
ALTER ICEBERG TABLE my_table CLUSTER BY (user_id, event_type);
```

Benefits:
- Related data co-located
- Better column statistics (tighter bounds)
- More effective file pruning

## Performance Checklist

| Check | Action |
|-------|--------|
| ☐ Partition pruning active? | Filter on partition columns |
| ☐ Files being skipped? | Add filters on columns with statistics |
| ☐ Many small files? | Run compaction |
| ☐ Wide column selection? | Select only needed columns |
| ☐ Join performance? | Ensure join keys have good distribution |

## Monitoring Queries

```sql
-- Query history with Iceberg-specific metrics
SELECT 
  query_id,
  query_text,
  partitions_scanned,
  partitions_total,
  bytes_scanned,
  rows_produced
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
WHERE query_type = 'SELECT'
ORDER BY start_time DESC;
```

## Best Practices

1. **Design partitions for query patterns**: Filter columns → partition columns
2. **Maintain file sizes**: 100MB-1GB optimal
3. **Regular compaction**: Prevent small file accumulation
4. **Use column statistics**: Filter on columns with good bounds
5. **Monitor pruning effectiveness**: Check partitions/files scanned
6. **Avoid SELECT ***: Request only needed columns
