# Data Lakehouse Architecture Patterns

## What is a Lakehouse?

A **data lakehouse** combines the best of data lakes and data warehouses:

| Data Lake | Data Warehouse | Lakehouse |
|-----------|----------------|-----------|
| Cheap storage | Expensive storage | Cheap storage |
| Schema-on-read | Schema-on-write | Schema evolution |
| No ACID | Full ACID | Full ACID |
| Poor performance | Great performance | Great performance |
| Raw data | Curated data | Both |

**Iceberg enables the lakehouse** by adding warehouse capabilities to lake storage.

## Core Lakehouse Principles

```
┌────────────────────────────────────────────────────────────┐
│                    LAKEHOUSE LAYER                         │
│  ┌─────────────────────────────────────────────────────┐  │
│  │              Apache Iceberg Tables                   │  │
│  │  • ACID transactions    • Schema evolution          │  │
│  │  • Time travel          • Multi-engine access       │  │
│  └─────────────────────────────────────────────────────┘  │
├────────────────────────────────────────────────────────────┤
│                    STORAGE LAYER                           │
│  ┌─────────────────────────────────────────────────────┐  │
│  │         Cloud Object Storage (S3/Azure/GCS)         │  │
│  │  • Parquet files        • Decoupled compute         │  │
│  │  • Unlimited scale      • Low cost                  │  │
│  └─────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

## Architecture Pattern: Medallion Architecture

### Bronze → Silver → Gold

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   BRONZE    │───►│   SILVER    │───►│    GOLD     │
│  Raw Data   │    │  Cleaned    │    │  Business   │
│             │    │  Conformed  │    │   Ready     │
└─────────────┘    └─────────────┘    └─────────────┘
     │                   │                   │
     ▼                   ▼                   ▼
  Iceberg            Iceberg            Iceberg
  Tables             Tables             Tables
```

### Implementation in Snowflake

```sql
-- Bronze: Raw ingestion (append-only)
CREATE ICEBERG TABLE bronze.events_raw (
  event_id STRING,
  payload VARIANT,
  ingested_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
)
CATALOG = 'SNOWFLAKE'
EXTERNAL_VOLUME = 'lakehouse_vol'
BASE_LOCATION = 'bronze/events_raw/';

-- Silver: Cleaned and typed
CREATE ICEBERG TABLE silver.events_cleaned (
  event_id BIGINT,
  event_type STRING,
  user_id BIGINT,
  event_time TIMESTAMP,
  properties MAP(STRING, STRING)
)
CATALOG = 'SNOWFLAKE'
EXTERNAL_VOLUME = 'lakehouse_vol'
BASE_LOCATION = 'silver/events_cleaned/'
PARTITION BY (DATE(event_time));

-- Gold: Business aggregates
CREATE ICEBERG TABLE gold.daily_event_summary (
  event_date DATE,
  event_type STRING,
  event_count BIGINT,
  unique_users BIGINT
)
CATALOG = 'SNOWFLAKE'
EXTERNAL_VOLUME = 'lakehouse_vol'
BASE_LOCATION = 'gold/daily_event_summary/'
PARTITION BY (event_date);
```

### Pipeline Flow

```sql
-- Bronze → Silver transformation
INSERT INTO silver.events_cleaned
SELECT 
  payload:event_id::BIGINT,
  payload:event_type::STRING,
  payload:user_id::BIGINT,
  payload:event_time::TIMESTAMP,
  payload:properties::MAP(STRING, STRING)
FROM bronze.events_raw
WHERE ingested_at > (SELECT MAX(event_time) FROM silver.events_cleaned);

-- Silver → Gold aggregation
MERGE INTO gold.daily_event_summary AS target
USING (
  SELECT 
    DATE(event_time) as event_date,
    event_type,
    COUNT(*) as event_count,
    COUNT(DISTINCT user_id) as unique_users
  FROM silver.events_cleaned
  WHERE DATE(event_time) = CURRENT_DATE() - 1
  GROUP BY 1, 2
) AS source
ON target.event_date = source.event_date 
   AND target.event_type = source.event_type
WHEN MATCHED THEN UPDATE SET 
  event_count = source.event_count,
  unique_users = source.unique_users
WHEN NOT MATCHED THEN INSERT VALUES (
  source.event_date, source.event_type, 
  source.event_count, source.unique_users
);
```

## Architecture Pattern: Hybrid Lakehouse

### Combine Snowflake Native + Iceberg

```
┌───────────────────────────────────────────────────────────┐
│                     SNOWFLAKE                             │
│  ┌─────────────────┐    ┌─────────────────────────────┐  │
│  │ Native Tables   │    │    Iceberg Tables           │  │
│  │ (Hot Data)      │    │    (Cold/Shared Data)       │  │
│  │                 │    │                             │  │
│  │ • Recent data   │    │ • Historical data           │  │
│  │ • High freq     │◄──►│ • Multi-engine access       │  │
│  │ • Internal only │    │ • Cost-optimized storage    │  │
│  └─────────────────┘    └─────────────────────────────┘  │
└───────────────────────────────────────────────────────────┘
```

### Implementation

```sql
-- Hot data in native Snowflake (last 30 days)
CREATE TABLE native.recent_events (
  event_id BIGINT,
  event_time TIMESTAMP,
  data VARIANT
) CLUSTER BY (DATE(event_time));

-- Cold data in Iceberg (historical)
CREATE ICEBERG TABLE iceberg.historical_events (
  event_id BIGINT,
  event_time TIMESTAMP,
  data VARIANT
)
CATALOG = 'SNOWFLAKE'
EXTERNAL_VOLUME = 'archive_vol'
BASE_LOCATION = 'historical_events/'
PARTITION BY (MONTH(event_time));

-- Unified view
CREATE VIEW analytics.all_events AS
SELECT * FROM native.recent_events
UNION ALL
SELECT * FROM iceberg.historical_events;

-- Scheduled archival
CREATE TASK archive_old_events
  WAREHOUSE = etl_wh
  SCHEDULE = 'USING CRON 0 1 * * * UTC'
AS
BEGIN
  INSERT INTO iceberg.historical_events
  SELECT * FROM native.recent_events
  WHERE event_time < DATEADD(day, -30, CURRENT_DATE());
  
  DELETE FROM native.recent_events
  WHERE event_time < DATEADD(day, -30, CURRENT_DATE());
END;
```

## Architecture Pattern: Multi-Cloud Lakehouse

### Unified Data Across Clouds

```
     AWS Region                     Azure Region
┌─────────────────────┐        ┌─────────────────────┐
│   S3 Bucket         │        │   Azure Storage     │
│   ┌─────────────┐   │        │   ┌─────────────┐   │
│   │  Iceberg    │   │        │   │  Iceberg    │   │
│   │  Tables     │   │        │   │  Tables     │   │
│   └─────────────┘   │        │   └─────────────┘   │
└─────────┬───────────┘        └─────────┬───────────┘
          │                              │
          └──────────────┬───────────────┘
                         │
                  ┌──────▼──────┐
                  │  Snowflake  │
                  │  (Query)    │
                  └─────────────┘
```

## Architecture Pattern: Real-Time Lakehouse

### Streaming + Batch Unified

```
Real-time Stream                    Batch Processing
      │                                   │
      ▼                                   ▼
┌──────────────┐                  ┌──────────────┐
│ Kafka/Kinesis│                  │  Batch ETL   │
│ + Iceberg    │                  │  (Spark)     │
│   Sink       │                  │              │
└──────┬───────┘                  └──────┬───────┘
       │                                 │
       └────────────┬────────────────────┘
                    │
            ┌───────▼────────┐
            │ Unified Iceberg│
            │     Table      │
            └───────┬────────┘
                    │
            ┌───────▼────────┐
            │   Snowflake    │
            │   (Query)      │
            └────────────────┘
```

## Best Practices for Lakehouse Architecture

### Data Organization

| Layer | Purpose | Retention | Access |
|-------|---------|-----------|--------|
| Bronze | Raw ingestion | Long | ETL only |
| Silver | Cleaned, typed | Medium | Analytics + ETL |
| Gold | Business-ready | Short/Cached | End users |

### Cost Optimization

1. **Use Iceberg for cold data**: Lower storage costs
2. **Partition by access patterns**: Enable efficient pruning
3. **Compact regularly**: Reduce small file overhead
4. **Expire snapshots**: Control metadata growth

### Performance

1. **Right-size layers**: Gold tables optimized for queries
2. **Pre-aggregate**: Reduce computation at query time
3. **Use native tables for hot paths**: When single-engine suffices
4. **Cache frequently accessed Gold tables**: Consider materialized views

### Governance

1. **Consistent naming**: `{layer}.{domain}_{object}`
2. **Lineage tracking**: Document transformations
3. **Access control**: RBAC per layer
4. **Data quality**: Validation at each layer boundary
