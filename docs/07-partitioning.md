# Partitioning Strategies & Hidden Partitioning

## Why Partition?

Partitioning organizes data files by column values, enabling **partition pruning**—queries that filter on partition columns only scan relevant files.

```
Without Partitioning:     With Partitioning (by date):
┌─────────────────┐      ┌─────────────────┐
│  All data files │      │ date=2024-01-01 │ ← Query scans only
│  (full scan)    │      ├─────────────────┤   matching partitions
│                 │      │ date=2024-01-02 │
└─────────────────┘      ├─────────────────┤
                         │ date=2024-01-03 │
                         └─────────────────┘
```

## Iceberg's Hidden Partitioning

Traditional partitioning (Hive-style) requires:
- Users to know partition scheme
- Queries to use exact partition column
- Data files in partition directories

**Iceberg's hidden partitioning:**
- Partition values derived from source columns via transforms
- Users query natural columns (e.g., `timestamp`)
- Iceberg automatically applies correct partition filter

### Example: Traditional vs. Hidden

```sql
-- Traditional (Hive): User must filter on partition column
SELECT * FROM events WHERE event_date = '2024-01-15';

-- Iceberg Hidden: Filter on source column, Iceberg handles partitioning
SELECT * FROM events WHERE event_timestamp >= '2024-01-15' 
                       AND event_timestamp < '2024-01-16';
-- Iceberg derives day partition from timestamp automatically
```

## Partition Transforms

| Transform | Input | Output | Example |
|-----------|-------|--------|---------|
| `identity` | Any | Same value | `region` → `region` |
| `year` | timestamp/date | Year int | `2024-01-15` → `2024` |
| `month` | timestamp/date | Year-Month | `2024-01-15` → `2024-01` |
| `day` | timestamp/date | Date | `2024-01-15 10:30` → `2024-01-15` |
| `hour` | timestamp | Date-Hour | `2024-01-15 10:30` → `2024-01-15-10` |
| `bucket(N)` | Any | Hash bucket 0 to N-1 | `bucket(16, user_id)` |
| `truncate(W)` | String/Int | Truncated value | `truncate(3, "abcdef")` → `"abc"` |

## Creating Partitioned Tables in Snowflake

### Basic Partitioning

```sql
CREATE ICEBERG TABLE events (
  event_id BIGINT,
  event_time TIMESTAMP,
  user_id BIGINT,
  event_type STRING,
  payload VARIANT
)
  CATALOG = 'SNOWFLAKE'
  EXTERNAL_VOLUME = 'my_vol'
  BASE_LOCATION = 'events/'
  PARTITION BY (event_type);
```

### Time-Based Partitioning

```sql
-- Partition by day (derived from timestamp)
CREATE ICEBERG TABLE events (
  event_id BIGINT,
  event_time TIMESTAMP,
  user_id BIGINT
)
  CATALOG = 'SNOWFLAKE'
  EXTERNAL_VOLUME = 'my_vol'
  BASE_LOCATION = 'events/'
  PARTITION BY (DATE(event_time));  -- day-level partition
```

### Multi-Column Partitioning

```sql
CREATE ICEBERG TABLE orders (
  order_id BIGINT,
  order_date DATE,
  region STRING,
  amount DECIMAL(10,2)
)
  CATALOG = 'SNOWFLAKE'
  EXTERNAL_VOLUME = 'my_vol'
  BASE_LOCATION = 'orders/'
  PARTITION BY (region, order_date);
```

### Bucket Partitioning

```sql
-- Distribute data evenly by user_id hash
CREATE ICEBERG TABLE user_events (
  user_id BIGINT,
  event_time TIMESTAMP,
  event_data VARIANT
)
  CATALOG = 'SNOWFLAKE'
  EXTERNAL_VOLUME = 'my_vol'
  BASE_LOCATION = 'user_events/'
  PARTITION BY (BUCKET(16, user_id), DATE(event_time));
```

## Partition Evolution

Iceberg supports **changing partition schemes without rewriting data**:

```
Old partition: MONTH(event_time)
New partition: DAY(event_time)

Old files: queried with month filter
New files: written with day partitions
```

Both old and new files remain queryable—Iceberg handles the mapping.

## Query Planning with Partitions

```
Query: WHERE event_time BETWEEN '2024-01-15' AND '2024-01-16'
              AND region = 'US'

Step 1: Partition Pruning
  - Filter: day(event_time) IN ['2024-01-15', '2024-01-16']
  - Filter: region = 'US'
  - Result: Only scan matching partition manifests

Step 2: File Pruning (within partitions)
  - Use column statistics (min/max) for additional filtering
```

## Partitioning Strategy Guidelines

| Data Pattern | Recommended Strategy |
|--------------|---------------------|
| Time-series data | `day(timestamp)` or `month(timestamp)` |
| High-cardinality ID | `bucket(N, id)` with N = cores × 2-4 |
| Regional/categorical | `identity(region)` |
| Composite patterns | Multi-level: `region, day(timestamp)` |

## Anti-Patterns

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| Too many partitions | Small files, slow planning | Coarser partitioning |
| High-cardinality identity | 1 file per value | Use `bucket()` |
| No partitioning on large tables | Full scans | Add time/category partition |
| Over-partitioning | Metadata bloat | Fewer partition columns |

## Partition Metrics

Check partition distribution:

```sql
-- View partition info (Snowflake)
SELECT 
  SYSTEM$CLUSTERING_INFORMATION('my_iceberg_table', '(partition_col)')
;
```

## Best Practices

1. **Match query patterns**: Partition on commonly filtered columns
2. **Target file size**: 100MB-1GB per partition per write
3. **Avoid high cardinality**: Use `bucket()` for unique IDs
4. **Time granularity**: Match to query patterns (daily queries → day partition)
5. **Evolve carefully**: Partition evolution is powerful but adds complexity
