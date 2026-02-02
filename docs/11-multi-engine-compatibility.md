# Multi-Engine Compatibility & Ecosystem Integration

## The Open Table Format Promise

Iceberg's primary value: **same data accessible by multiple engines** without conversion or copying.

```
                    ┌─────────────┐
                    │   Iceberg   │
                    │   Table     │
                    │  (S3/Azure) │
                    └──────┬──────┘
                           │
       ┌───────────────────┼───────────────────┐
       │                   │                   │
       ▼                   ▼                   ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Snowflake  │    │   Spark     │    │   Trino     │
└─────────────┘    └─────────────┘    └─────────────┘
       │                   │                   │
       └───────────────────┴───────────────────┘
                           │
                    All engines see
                    consistent data
```

## Compatible Engines

### Query Engines

| Engine | Read | Write | Catalog Support |
|--------|------|-------|-----------------|
| **Snowflake** | ✓ | ✓ | Snowflake, Glue, REST |
| **Apache Spark** | ✓ | ✓ | All catalogs |
| **Trino/Presto** | ✓ | ✓ | All catalogs |
| **Apache Flink** | ✓ | ✓ | All catalogs |
| **Dremio** | ✓ | ✓ | All catalogs |
| **Amazon Athena** | ✓ | ✓ | Glue |
| **Amazon EMR** | ✓ | ✓ | Glue, Hive |
| **Databricks** | ✓ | ✓ | Unity Catalog |
| **Starburst** | ✓ | ✓ | All catalogs |
| **DuckDB** | ✓ | Limited | Direct metadata |

### Data Integration Tools

| Tool | Read | Write |
|------|------|-------|
| **Apache Kafka + Iceberg Sink** | - | ✓ |
| **Apache Airflow** | Via engines | Via engines |
| **dbt** | Via engines | Via engines |
| **Fivetran** | - | ✓ |

## Catalog Interoperability

### Shared Catalog Pattern

Multiple engines access same catalog:

```
          ┌─────────────────┐
          │   AWS Glue      │
          │   Data Catalog  │
          └────────┬────────┘
                   │
    ┌──────────────┼──────────────┐
    │              │              │
    ▼              ▼              ▼
Snowflake       Spark        Athena
(catalog        (native      (native
integration)    Glue)        Glue)
```

### REST Catalog (Iceberg REST Spec)

Universal catalog protocol:

```
          ┌─────────────────┐
          │  REST Catalog   │
          │ (Polaris, etc.) │
          └────────┬────────┘
                   │
    ┌──────────────┼──────────────┐
    │              │              │
    ▼              ▼              ▼
Snowflake       Spark         Trino
```

## Snowflake as Multi-Engine Participant

### Reading External Iceberg Tables

```sql
-- Connect to Glue-managed tables
CREATE CATALOG INTEGRATION glue_int
  CATALOG_SOURCE = GLUE
  CATALOG_NAMESPACE = 'my_database'
  TABLE_FORMAT = ICEBERG
  GLUE_AWS_ROLE_ARN = 'arn:aws:iam::123:role/role'
  GLUE_CATALOG_ID = '123456789'
  ENABLED = TRUE;

-- Access table written by Spark
CREATE ICEBERG TABLE spark_table
  EXTERNAL_VOLUME = 'my_vol'
  CATALOG = 'glue_int'
  CATALOG_TABLE_NAME = 'spark_written_table';

SELECT * FROM spark_table;
```

### Writing to Shared Tables

```sql
-- Write to externally managed table
INSERT INTO spark_table 
SELECT * FROM snowflake_source;

-- Changes visible to all engines immediately
```

## Cross-Engine Patterns

### Pattern 1: Snowflake Query, Spark ETL

```
Spark (ETL) ──► Write ──► Iceberg ──► Read ──► Snowflake (Analytics)
```

Use case: Heavy transformations in Spark, fast queries in Snowflake

### Pattern 2: Snowflake Primary, Multi-Engine Read

```
Snowflake ──► Write ──► Iceberg ──► Read ──► Spark ML
                           │
                           └──► Read ──► Trino Dashboards
```

Use case: Snowflake as source of truth, specialized engines for specific tasks

### Pattern 3: Federated Analytics

```
Source A ──► Spark ──► Iceberg Table A ◄──┐
                                          │
Source B ──► Flink ──► Iceberg Table B ◄──┼── Snowflake (unified query)
                                          │
Source C ──► Trino ──► Iceberg Table C ◄──┘
```

## Compatibility Considerations

### Schema Compatibility

| Issue | Mitigation |
|-------|-----------|
| Type mapping differences | Use common types (long, string, timestamp) |
| Nested type support | Varies by engine; test complex types |
| Decimal precision | Align precision/scale across engines |

### Feature Compatibility

| Feature | Support Varies |
|---------|---------------|
| Schema evolution | Generally consistent |
| Partition evolution | Check engine version |
| Row-level deletes | v2 spec required |
| Merge-on-read | Not all engines |
| Hidden partitioning | Most engines |

### Catalog Sync

| Scenario | Solution |
|----------|----------|
| Metadata out of sync | Refresh table metadata |
| Schema changes not visible | Re-sync catalog integration |
| New partitions not seen | Refresh partition metadata |

```sql
-- Refresh external table metadata in Snowflake
ALTER ICEBERG TABLE my_external_table REFRESH;
```

## Version Compatibility

### Iceberg Table Format Versions

| Version | Features |
|---------|----------|
| **v1** | Basic operations, append-only |
| **v2** | Row-level deletes, update/delete/merge |

```sql
-- Check table format version
SELECT * FROM TABLE(INFORMATION_SCHEMA.TABLES)
WHERE table_name = 'MY_TABLE';
```

### Engine Version Requirements

Ensure all engines support same Iceberg version:
- Spark 3.x: Iceberg 0.13+
- Trino 400+: Iceberg v2
- Snowflake: Iceberg v2 (current)

## Best Practices

1. **Standardize on catalog**: Single catalog reduces sync issues
2. **Test cross-engine**: Verify read/write compatibility before production
3. **Document engine matrix**: Track which engines access which tables
4. **Coordinate schema changes**: Notify all engine users of changes
5. **Monitor for conflicts**: Multi-writer scenarios need coordination
6. **Use common data types**: Avoid engine-specific type extensions
