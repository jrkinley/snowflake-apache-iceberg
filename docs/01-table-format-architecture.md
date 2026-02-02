# Apache Iceberg Table Format Architecture

## Overview

Apache Iceberg is an open table format designed for large-scale analytic datasets. The specification defines a **metadata tree structure** that tracks data files through a hierarchy of metadata objects, with an Iceberg catalog providing table discovery.

## Iceberg Metadata Tree Structure

The spec defines this hierarchy from catalog to data:

```
┌─────────────────────────────────────────────────────────────────┐
│  ICEBERG CATALOG                                                │
│  Stores "current metadata pointer" for each table               │
│  (Snowflake Horizon, Polaris, Unity Catalog, AWS Glue)          │
└───────────────────────────┬─────────────────────────────────────┘
                            │ points to
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  METADATA FILE (JSON)                                           │
│  Contains: schemas, partition specs, current-snapshot-id,       │
│  snapshot history, sort orders, table properties                │
└───────────────────────────┬─────────────────────────────────────┘
                            │ snapshot references
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  MANIFEST LIST (Avro)                                           │
│  One per snapshot. Lists manifest files with partition summaries│
└───────────────────────────┬─────────────────────────────────────┘
                            │ lists
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  MANIFEST FILES (Avro)                                          │
│  Track data files with: path, partition values, column stats    │
│  (min/max bounds, null counts, row counts)                      │
└───────────────────────────┬─────────────────────────────────────┘
                            │ track
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  DATA FILES (Parquet)                                           │
│  Immutable columnar files in cloud object storage               │
│  Note: Snowflake supports Parquet only; Iceberg spec also       │
│  supports ORC and Avro                                          │
└─────────────────────────────────────────────────────────────────┘
```

## Spec Components Explained

### Iceberg Catalog

The catalog is responsible for:
- **Table discovery**: Map table identifiers (db.schema.table) to metadata file locations
- **Atomic commits**: Ensure metadata pointer updates are atomic (compare-and-swap)
  - Prevents concurrent writers from overwriting each other's changes, guaranteeing table consistency
- **Namespace management**: Organize tables into databases/schemas

**Catalog implementations** (Snowflake-compatible):

| Catalog | Type | Snowflake Integration |
|---------|------|----------------------|
| **Snowflake Horizon** | Native | `CATALOG = 'SNOWFLAKE'` |
| **Polaris** | REST (Snowflake OSS) | `CATALOG_SOURCE = ICEBERG_REST` |
| **AWS Glue** | AWS-managed | `CATALOG_SOURCE = GLUE` |
| **Databricks Unity Catalog** | REST | `CATALOG_SOURCE = ICEBERG_REST` |

### Metadata File

JSON file containing the complete table definition:

| Field | Description |
|-------|-------------|
| `format-version` | Iceberg spec version (1 or 2) |
| `table-uuid` | Unique table identifier |
| `schemas` | List of schema versions |
| `partition-specs` | Partition specifications |
| `current-snapshot-id` | Pointer to current snapshot |
| `snapshots` | List of all snapshots |

> **Metadata file management**: The `snapshots` array grows with each commit, enabling Time Travel queries and UNDROP operations. Without cleanup, this file becomes bloated. **Snowflake-managed tables**: Snowflake automatically expires snapshots and deletes old metadata based on `DATA_RETENTION_TIME_IN_DAYS` (default 1 day). **External catalog tables**: The external catalog or tooling manages cleanup—Snowflake does not delete external metadata.

### Manifest List

Avro file (one per snapshot) that:
- Lists all manifest files for that snapshot
- Contains partition summary statistics for quick pruning
- Tracks which manifests were added/removed

> **Manifest list management**: Each snapshot references exactly one manifest list file. When a snapshot expires, its manifest list becomes orphaned. **Snowflake-managed tables**: Snowflake deletes orphaned manifest lists 7-14 days after the snapshot expires. **External catalog tables**: Cleanup requires external tooling.

### Manifest Files

Avro files that track **multiple data files** (not 1:1). Each manifest file contains entries for many data files, typically grouped by partition or write operation:
- File path and format
- Partition tuple values
- Column-level statistics (bounds, nulls, counts)
- File size and row count

> **Manifest file management**: Manifest files can be shared across snapshots — if data files haven't changed, the manifest is reused. A manifest file becomes orphaned only when no manifest list references it. **Snowflake-managed tables**: Snowflake deletes orphaned manifest files 7-14 days after all referencing snapshots expire. **External catalog tables**: Cleanup requires external tooling.

### Data Files

Immutable columnar Parquet files:
- **Never modified in place** — immutability enables snapshot isolation and consistent reads; concurrent queries always see complete, valid files
- **Changes create new files** — when data is updated or deleted, Iceberg supports two modes:
  - **Copy-on-write (default)**: Rewrites entire affected data files with changes applied; slower writes but faster reads
  - **Merge-on-read**: Writes small delete files marking changed rows; faster writes but reads must merge delete files with data files at query time. Compaction rewrites data files with delete files applied, restoring read performance. **Snowflake-managed tables**: Compaction runs automatically in the background, creating a new snapshot with optimized files
- **Written to cloud object storage** (S3, Azure Blob, GCS)
- **Organized by partition values** — files are grouped into directories based on partition columns to enable efficient pruning

**Partition strategies** (using Iceberg transforms):

| Strategy | Transform | Example | Use Case |
|----------|-----------|---------|----------|
| Date/Time | `day(ts)`, `month(ts)`, `year(ts)` | `order_date=2024-01-15/` | Time-series data, daily/monthly reporting |
| Region/Category | `identity(col)` | `region=US/` | Geographic or categorical filtering |
| Combined | Multiple transforms | `region=US/order_date=2024-01-15/` | Multi-dimensional filtering |
| Hour-level | `hour(ts)` | `hour=2024-01-15-10/` | High-volume streaming data |
| Bucket | `bucket(N, col)` | `user_id_bucket=7/` | Distributes high-cardinality columns (e.g., `user_id`) evenly across N buckets |
| Truncate | `truncate(W, col)` | `sku_prefix=ABC/` | Groups strings/integers by prefix width W |

> **Snowflake Note**: Snowflake supports **Parquet only**. The Iceberg spec also supports ORC and Avro, but these are not available when using Snowflake as the query engine.

> **Data file management**: Data files are deleted when no manifest file references them (and by extension, no snapshot). A data file becomes orphaned when all snapshots containing manifest files that reference it have expired. **Snowflake-managed tables**: Snowflake deletes orphaned data files 7-14 days after the last referencing snapshot expires. **External catalog tables**: Cleanup requires external tooling — Snowflake does not delete data files for externally managed tables.

## Key Design Principles

| Principle | Implementation |
|-----------|----------------|
| **Immutability** | Data files are never modified; new files are written |
| **Snapshot Isolation** | Each query sees a consistent snapshot |
| **File-Level Tracking** | Each file tracked individually with statistics |
| **Serializable Isolation** | Concurrent transactions appear to execute one after another; achieved via optimistic concurrency using Compare-And-Swap (CAS) on the catalog's metadata pointer (the reference to the current metadata file location) |

> **Optimistic Concurrency Control (OCC)**: Apache Iceberg uses OCC for transaction management. OCC assumes conflicts are rare — transactions proceed without locks, then validate at commit time. If a conflict is detected (another writer committed first), the transaction retries. Iceberg uses OCC because it suits cloud object storage where locks are impractical.

## Why This Architecture Matters

| Benefit | How Iceberg Achieves It |
|---------|------------------------|
| **No directory listing** | Manifest files enumerate data files explicitly |
| **Atomic commits** | Catalog performs atomic metadata pointer swap |
| **Engine agnostic** | Open spec — any engine can read/write |
| **Cloud native** | Designed for object storage where directory listing is slow/expensive and locks are impractical; Iceberg avoids listings via manifest files and uses OCC instead of locks |
| **Efficient pruning** | Statistics at manifest level enable skipping |

## Snowflake Implementation

### Snowflake-Managed Iceberg Tables (Horizon Catalog)

When using `CATALOG = 'SNOWFLAKE'`, Snowflake's **Horizon Catalog** acts as the Iceberg catalog. This is referred to as **Snowflake-managed Iceberg tables**.

**What Snowflake manages:**
- **Catalog operations**: Metadata pointer management, atomic commits, table discovery
- **Metadata lifecycle**: Automatic cleanup of expired snapshots, manifest lists, and manifest files. Snapshots expire based on `DATA_RETENTION_TIME_IN_DAYS` (default: 1 day, configurable at account/database/schema/table level). Snowflake performs the actual file deletion 7-14 days after expiration (this delay is not configurable)
- **Data compaction**: Background compaction for merge-on-read tables
- **File organization**: Metadata files (JSON) stored in `BASE_LOCATION/metadata/`, data files (Parquet) in `BASE_LOCATION/data/`

**Snowflake query optimizations:**
- **Micro-partition pruning**: Snowflake uses Iceberg manifest statistics to skip irrelevant files before scanning
- **Predicate pushdown**: Filters applied at the storage layer, reducing data read from object storage
- **Parallel execution**: Files scanned concurrently across Snowflake's distributed compute
- **Result caching**: Repeated queries can leverage Snowflake's query result cache

**Business value:**
- Single platform for governance, security, and query execution
- No external catalog infrastructure to manage
- Snowflake handles all table maintenance automatically
- Full SQL support including INSERT, UPDATE, DELETE, MERGE

### Example: OpenTelemetry Traces Table

```sql
CREATE OR REPLACE EXTERNAL VOLUME otel_traces_vol
STORAGE_LOCATIONS = ((
    NAME            = 's3_otel_data'
    STORAGE_PROVIDER = 'S3'
    STORAGE_BASE_URL = 's3://my-data-lake/otel/'
    STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::123456789012:role/snowflake-iceberg-role'
));

CREATE OR REPLACE ICEBERG TABLE otel_traces (
    trace_id            VARCHAR(32)   NOT NULL,
    span_id             VARCHAR(16)   NOT NULL,
    parent_span_id      VARCHAR(16),
    trace_state         VARCHAR(512),
    span_name           VARCHAR(256)  NOT NULL,
    span_kind           VARCHAR(20),
    start_time          TIMESTAMP_NTZ NOT NULL,
    end_time            TIMESTAMP_NTZ NOT NULL,
    duration_ms         INT,
    status_code         VARCHAR(10),
    status_message      VARCHAR(1024),
    service_name        VARCHAR(256)  NOT NULL,
    service_namespace   VARCHAR(256),
    service_version     VARCHAR(64),
    resource_attributes OBJECT,
    span_attributes     OBJECT,
    events              ARRAY,
    links               ARRAY
)
CATALOG         = 'SNOWFLAKE'
EXTERNAL_VOLUME = 'otel_traces_vol'
BASE_LOCATION   = 'traces/'
PARTITION BY (DATE(start_time), service_name);
```

## Further Reading

- [Apache Iceberg Spec](https://iceberg.apache.org/spec/) - Official table format specification
- [Snowflake Iceberg Tables](https://docs.snowflake.com/en/user-guide/tables-iceberg) - Snowflake implementation docs
