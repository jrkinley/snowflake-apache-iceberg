# Parquet Integration & Data Storage

## Why Parquet?

Apache Iceberg primarily uses **Apache Parquet** as the underlying file format. Parquet is a columnar storage format optimized for analytical workloads.

## Parquet File Structure

```
┌────────────────────────────────────┐
│           Row Group 1              │
│  ┌──────┬──────┬──────┬──────┐    │
│  │Col A │Col B │Col C │Col D │    │
│  │chunk │chunk │chunk │chunk │    │
│  └──────┴──────┴──────┴──────┘    │
├────────────────────────────────────┤
│           Row Group 2              │
│  ┌──────┬──────┬──────┬──────┐    │
│  │Col A │Col B │Col C │Col D │    │
│  │chunk │chunk │chunk │chunk │    │
│  └──────┴──────┴──────┴──────┘    │
├────────────────────────────────────┤
│            Footer                  │
│  (schema, row group metadata)      │
└────────────────────────────────────┘
```

## Key Parquet Features for Iceberg

| Feature | Benefit |
|---------|---------|
| **Columnar Storage** | Read only columns needed for query |
| **Compression** | Snappy, Zstd, Gzip reduce storage costs |
| **Encoding** | Dictionary, RLE, delta encoding for efficiency |
| **Statistics** | Min/max per column chunk enables predicate pushdown |
| **Nested Types** | Native support for structs, arrays, maps |

## Iceberg + Parquet Integration

### Column Statistics Flow

```
Parquet File Stats → Manifest File → Query Planning
     (per file)        (aggregated)    (file pruning)
```

Iceberg captures Parquet statistics in manifest files:
- **Min/Max values** per column
- **Null counts**
- **Row counts**
- **File size**

### Data File Naming

Iceberg generates unique file names to prevent collisions:

```
data/
├── 00000-0-a1b2c3d4-e5f6-7890-abcd-ef1234567890.parquet
├── 00001-0-b2c3d4e5-f6a7-8901-bcde-f12345678901.parquet
└── 00002-0-c3d4e5f6-a7b8-9012-cdef-123456789012.parquet
```

## External Volume Configuration (Snowflake)

### Amazon S3

```sql
CREATE EXTERNAL VOLUME iceberg_vol
  STORAGE_LOCATIONS = (
    (
      NAME = 's3_location'
      STORAGE_PROVIDER = 'S3'
      STORAGE_BASE_URL = 's3://my-bucket/iceberg/'
      STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::123456789:role/iceberg-role'
    )
  );
```

### Azure Blob Storage

```sql
CREATE EXTERNAL VOLUME iceberg_vol
  STORAGE_LOCATIONS = (
    (
      NAME = 'azure_location'
      STORAGE_PROVIDER = 'AZURE'
      STORAGE_BASE_URL = 'azure://container@account.blob.core.windows.net/iceberg/'
      AZURE_TENANT_ID = 'tenant-id'
    )
  );
```

### Google Cloud Storage

```sql
CREATE EXTERNAL VOLUME iceberg_vol
  STORAGE_LOCATIONS = (
    (
      NAME = 'gcs_location'
      STORAGE_PROVIDER = 'GCS'
      STORAGE_BASE_URL = 'gcs://my-bucket/iceberg/'
    )
  );
```

## File Size Considerations

| Scenario | Recommended File Size |
|----------|----------------------|
| **Streaming ingestion** | 64-128 MB |
| **Batch loads** | 128-512 MB |
| **Large scans** | 256 MB - 1 GB |

Set target file size in Snowflake:

```sql
CREATE ICEBERG TABLE my_table (...)
  TARGET_FILE_SIZE = '128MB';
```

## Storage Layout

```
external_volume_location/
└── base_location/
    ├── data/
    │   ├── partition=value1/
    │   │   └── *.parquet
    │   └── partition=value2/
    │       └── *.parquet
    └── metadata/
        ├── v1.metadata.json
        ├── snap-*.avro
        └── *.avro
```

## Best Practices

1. **Right-size files**: Avoid many small files (slow listing) or huge files (poor parallelism)
2. **Use compression**: Zstd offers best compression/speed tradeoff
3. **Co-locate data**: Keep data in same region as compute
4. **Consider partitioning**: Reduces data scanned for filtered queries
