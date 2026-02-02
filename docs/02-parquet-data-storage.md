# Parquet Integration & Data Storage

## Why Parquet?

**Snowflake supports Apache Parquet exclusively** for Iceberg tables. While the Iceberg specification also supports ORC and Avro, Snowflake's Iceberg implementation is Parquet-only.

### What is Apache Parquet?

Apache Parquet is an open-source, **column-oriented data file format** designed for efficient data storage and retrieval. It provides high-performance compression and encoding schemes to handle complex data in bulk.

**Key characteristics:**
- **Origin**: Created to bring compressed, efficient columnar data representation to the Hadoop ecosystem
- **Nested data support**: Built from the ground up for complex nested structures using the record shredding and assembly algorithm from Google's [Dremel paper](https://research.google/pubs/pub36632/)
- **Per-column compression**: Compression schemes can be specified at the column level, optimizing for each column's data characteristics
- **Language agnostic**: Supported across many programming languages (Java, Python, C++, etc.) and analytics tools (Snowflake, Spark, Trino, etc.)

### Row vs Columnar Storage

| Aspect | Row-Oriented (e.g., Avro, RDBMS) | Columnar (Parquet) |
|--------|--------------------------------|-------------------|
| **Storage layout** | Data stored row-by-row | Data stored column-by-column |
| **Read pattern** | Must read entire row | Read only columns needed |
| **Compression** | Limited (varied data types per row) | Excellent (same data type per column) |
| **Best for** | OLTP, single-row lookups | OLAP, analytical aggregations |

### Why Columnar is Optimized for Analytics

1. **I/O efficiency**: Analytical queries typically select few columns from wide tables — columnar reads only what's needed
2. **Better compression**: Same-type values compress significantly better (e.g., all integers, all timestamps)
3. **Vectorized processing**: CPUs process batches of same-type values more efficiently
4. **Predicate pushdown**: Column statistics (min/max) enable skipping irrelevant data blocks

## Parquet File Structure

A Parquet file consists of three main components:

| Component | Description |
|-----------|-------------|
| **Row Group** | Horizontal partition containing a subset of rows (typically 128MB-1GB). Enables parallel processing — each row group can be read independently |
| **Column Chunk** | Vertical slice within a row group containing all values for one column. Each chunk is compressed and encoded separately |
| **Footer** | File metadata containing: schema definition, row group byte offsets (enabling direct seeks without scanning), column statistics (min/max/null counts), and encoding/compression info |

> **Read path**: Query engines read the footer first to understand file structure, then selectively read only the column chunks needed based on query columns and filter predicates.

```
┌─────────────────────────────────────────────────────┐
│                     ROW GROUP 1                     │
│  ┌───────────┬───────────┬───────────┬───────────┐  │
│  │  Col A    │  Col B    │  Col C    │  Col D    │  │
│  │  chunk    │  chunk    │  chunk    │  chunk    │  │
│  └───────────┴───────────┴───────────┴───────────┘  │
├─────────────────────────────────────────────────────┤
│                     ROW GROUP 2                     │
│  ┌───────────┬───────────┬───────────┬───────────┐  │
│  │  Col A    │  Col B    │  Col C    │  Col D    │  │
│  │  chunk    │  chunk    │  chunk    │  chunk    │  │
│  └───────────┴───────────┴───────────┴───────────┘  │
├─────────────────────────────────────────────────────┤
│                       FOOTER                        │
│       (schema, row group byte offsets,              │
│        column statistics, encoding info)            │
└─────────────────────────────────────────────────────┘
```

## Key Parquet Features for Iceberg

| Feature | Benefit |
|---------|---------|
| **Columnar Storage** | Read only columns needed for query |
| **Compression** | Reduces file size on disk (Snappy, Zstd, Gzip) |
| **Encoding** | Reduces data size before compression (Dictionary, RLE, Delta) |
| **Statistics** | Enables skipping irrelevant data blocks (predicate pushdown — filtering at storage layer, not in memory) |
| **Nested Types** | Native support for structs, arrays, maps |

## File Size Considerations

File size is a tradeoff between write efficiency and read efficiency:

- **Small files (< 64 MB)**: Faster commits, better for frequent updates, finer parallelism — but more metadata overhead and slower query planning
- **Large files (> 1 GB)**: Less metadata, fewer file operations, better compression — but slower commits and more data to rewrite on updates

**Recommended sizes by use case:**

| Scenario | Recommended Size | Why |
|----------|------------------|-----|
| **Streaming ingestion** | 64-128 MB | Balance between commit latency and metadata overhead |
| **Batch loads** | 128-512 MB | Larger files for better compression; fewer commits |
| **Large scans** | 256 MB - 1 GB | Maximizes I/O efficiency for full table scans |

Set target file size in Snowflake:

```sql
CREATE ICEBERG TABLE my_table (...)
  TARGET_FILE_SIZE = '128MB';
```

## Best Practices

### File Management
- **Right-size files** (64 MB - 1 GB): Avoid small file problem (metadata overhead, slow planning) and oversized files (poor parallelism) — *Apache Iceberg & Snowflake*
- **Use Zstd compression**: Best compression-to-speed ratio for most workloads — *Apache Iceberg*
- **Run compaction regularly**: Merge small files created by streaming ingestion — *Snowflake: automatic for managed tables*

### Data Organization
- **Partition by query patterns**: Choose partition columns based on common filter predicates (e.g., date, region) — *Apache Iceberg*
- **Avoid over-partitioning**: Too many partitions creates small files; aim for 100MB+ per partition — *Apache Iceberg*
- **Order columns strategically**: Place frequently filtered columns first for better statistics effectiveness — *Apache Parquet*

### Infrastructure
- **Co-locate storage and compute**: Keep external volume in same cloud region as Snowflake warehouse to minimize latency — *Snowflake*
- **Use appropriate storage class**: Standard storage for frequently accessed data; consider lifecycle policies for cold data — *Snowflake*

### Maintenance
- **Expire old snapshots**: Prevent unbounded metadata growth with `DATA_RETENTION_TIME_IN_DAYS` — *Snowflake*
- **Monitor file counts**: High file counts per partition indicate need for compaction — *Apache Iceberg*
