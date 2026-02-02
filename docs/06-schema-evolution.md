# Schema Evolution & Data Types

## Schema Evolution Overview

Iceberg supports **safe schema evolution** without rewriting data files. Changes are tracked in metadata, and data files are read with the appropriate schema version.

## Supported Schema Changes

| Operation | Description | Data Rewrite? |
|-----------|-------------|---------------|
| **Add Column** | Add new column (any position) | No |
| **Drop Column** | Remove column from schema | No |
| **Rename Column** | Change column name | No |
| **Reorder Columns** | Change column position | No |
| **Widen Type** | int → long, float → double | No |
| **Make Nullable** | required → optional | No |

## Schema Evolution in Snowflake

### Add Column

```sql
ALTER ICEBERG TABLE my_table 
  ADD COLUMN new_column STRING;

-- Add with position
ALTER ICEBERG TABLE my_table 
  ADD COLUMN new_column STRING AFTER existing_column;
```

### Drop Column

```sql
ALTER ICEBERG TABLE my_table 
  DROP COLUMN old_column;
```

### Rename Column

```sql
ALTER ICEBERG TABLE my_table 
  RENAME COLUMN old_name TO new_name;
```

### Change Data Type

```sql
-- Type widening only
ALTER ICEBERG TABLE my_table 
  ALTER COLUMN my_int SET DATA TYPE BIGINT;
```

## How Schema Evolution Works

### Column IDs (Not Names)

Iceberg uses **column IDs** internally, not names:

```json
{
  "schema": {
    "fields": [
      {"id": 1, "name": "id", "type": "long"},
      {"id": 2, "name": "name", "type": "string"},
      {"id": 3, "name": "email", "type": "string"}
    ]
  }
}
```

- Renaming column changes name but keeps ID
- Old data files still readable via ID mapping
- Dropped columns: ID reserved, not reused

### Schema History

```
Schema v1: [id:1, name:2, email:3]
     │
     ▼ ADD COLUMN phone
Schema v2: [id:1, name:2, email:3, phone:4]
     │
     ▼ DROP COLUMN email
Schema v3: [id:1, name:2, phone:4]
     │
     ▼ RENAME name → full_name
Schema v4: [id:1, full_name:2, phone:4]
```

Data files written with Schema v1 are still readable with Schema v4.

## Iceberg Data Types

### Primitive Types

| Iceberg Type | Snowflake Type | Description |
|--------------|----------------|-------------|
| `boolean` | BOOLEAN | True/false |
| `int` | INTEGER | 32-bit signed |
| `long` | BIGINT | 64-bit signed |
| `float` | FLOAT | 32-bit IEEE 754 |
| `double` | DOUBLE | 64-bit IEEE 754 |
| `decimal(P,S)` | NUMBER(P,S) | Arbitrary precision |
| `date` | DATE | Calendar date |
| `time` | TIME | Time without timezone |
| `timestamp` | TIMESTAMP_NTZ | Microsecond precision |
| `timestamptz` | TIMESTAMP_TZ | With timezone |
| `string` | VARCHAR | UTF-8 string |
| `binary` | BINARY | Byte array |
| `uuid` | VARCHAR(36) | UUID string |

### Complex Types

| Iceberg Type | Description |
|--------------|-------------|
| `struct<...>` | Named fields (like object) |
| `list<element>` | Ordered collection |
| `map<key, value>` | Key-value pairs |

### Nested Type Example

```sql
CREATE ICEBERG TABLE events (
  event_id BIGINT,
  event_time TIMESTAMP,
  user OBJECT(
    id BIGINT,
    name STRING,
    tags ARRAY(STRING)
  ),
  properties MAP(STRING, STRING)
)
CATALOG = 'SNOWFLAKE'
EXTERNAL_VOLUME = 'my_vol'
BASE_LOCATION = 'events/';
```

## Type Promotion Rules

Safe type promotions (no data loss):

```
int     → long
float   → double
decimal → decimal (wider precision/scale)
```

**Not Allowed:**
- Narrowing conversions (long → int)
- Type changes (string → int)
- Precision reduction

## Schema Evolution vs. Standard Tables

| Aspect | Standard Snowflake | Iceberg Table |
|--------|-------------------|---------------|
| Add column | Instant | Instant (metadata only) |
| Drop column | Instant | Instant (metadata only) |
| Rename | Instant | Instant (ID preserved) |
| Type change | Limited | Type widening only |
| Historical data | No impact | Readable via ID mapping |

## Best Practices

1. **Plan schema carefully**: Avoid frequent changes
2. **Use nullable columns**: Easier evolution (add optional fields)
3. **Column naming**: IDs are stable; names can change
4. **Type selection**: Start with wider types if growth expected
5. **Test compatibility**: Verify consumers handle schema changes
6. **Document changes**: Track schema evolution for data lineage
