# Catalog Systems & Table Discovery

## The Catalog's Role

The catalog is the **entry point** for Iceberg tables. It maps table identifiers (database.schema.table) to the current metadata file location.

```
Client Query: "SELECT * FROM db.schema.my_table"
     │
     ▼
┌─────────────┐     ┌──────────────────┐     ┌─────────────┐
│   Catalog   │ ──► │ Metadata Pointer │ ──► │ Table Data  │
└─────────────┘     └──────────────────┘     └─────────────┘
```

## Catalog Responsibilities

| Function | Description |
|----------|-------------|
| **Namespace Management** | Organize tables into databases/schemas |
| **Table Discovery** | Resolve table names to metadata locations |
| **Atomic Commits** | Ensure metadata pointer updates are atomic |
| **Access Control** | Enforce permissions (catalog-dependent) |

## Snowflake Catalog Options

### Option 1: Snowflake-Managed (Horizon Catalog)

Snowflake acts as the Iceberg catalog with full read/write capabilities.

```sql
CREATE ICEBERG TABLE customer (
  id INT,
  name STRING,
  email STRING
)
  CATALOG = 'SNOWFLAKE'
  EXTERNAL_VOLUME = 'my_volume'
  BASE_LOCATION = 'customer/';
```

**Characteristics:**
- Full DML support (INSERT, UPDATE, DELETE, MERGE)
- Snowflake manages metadata lifecycle
- Automatic statistics collection
- Time travel via Snowflake's snapshot retention
- Integrated with Snowflake RBAC

**Use Cases:**
- New Iceberg implementations
- Snowflake-primary workloads
- Need full transactional support

### Option 2: External Catalog Integration

Connect to tables managed by external catalogs (AWS Glue, REST catalogs).

#### AWS Glue Catalog

```sql
-- Create catalog integration
CREATE CATALOG INTEGRATION glue_catalog
  CATALOG_SOURCE = GLUE
  CATALOG_NAMESPACE = 'my_glue_database'
  TABLE_FORMAT = ICEBERG
  GLUE_AWS_ROLE_ARN = 'arn:aws:iam::123456789:role/glue-role'
  GLUE_CATALOG_ID = '123456789'
  ENABLED = TRUE;

-- Create table from Glue catalog
CREATE ICEBERG TABLE my_table
  EXTERNAL_VOLUME = 'my_volume'
  CATALOG = 'glue_catalog'
  CATALOG_TABLE_NAME = 'source_table';
```

#### Iceberg REST Catalog

```sql
-- Create REST catalog integration
CREATE CATALOG INTEGRATION rest_catalog
  CATALOG_SOURCE = ICEBERG_REST
  TABLE_FORMAT = ICEBERG
  CATALOG_URI = 'https://my-catalog.example.com/api'
  WAREHOUSE = 'my_warehouse'
  ENABLED = TRUE;

-- Create table from REST catalog
CREATE ICEBERG TABLE my_table
  EXTERNAL_VOLUME = 'my_volume'
  CATALOG = 'rest_catalog'
  CATALOG_TABLE_NAME = 'my_namespace.my_table';
```

## Catalog-Linked Databases

For tighter integration, create a database linked to an external catalog:

```sql
CREATE DATABASE my_glue_db
  CATALOG = 'glue_catalog';

USE DATABASE my_glue_db;

-- Tables created here appear in both Snowflake and Glue
CREATE ICEBERG TABLE new_table (
  id INT,
  value STRING
);
```

## Comparison: Snowflake vs External Catalogs

| Feature | Snowflake-Managed | External Catalog |
|---------|-------------------|------------------|
| **Write Support** | Full DML | Full DML (REST v2) |
| **Catalog Control** | Snowflake | External system |
| **Multi-Engine Access** | Via Iceberg REST | Native |
| **Schema Management** | Snowflake | External catalog |
| **Snapshot Retention** | Snowflake-controlled | External-controlled |
| **Access Control** | Snowflake RBAC | Catalog-dependent |

## External Catalog Write Support

Snowflake supports **writes to externally managed tables** with Iceberg REST catalogs:

```sql
-- External catalog with write support
CREATE CATALOG INTEGRATION writable_catalog
  CATALOG_SOURCE = ICEBERG_REST
  TABLE_FORMAT = ICEBERG
  CATALOG_URI = 'https://polaris.example.com'
  WAREHOUSE = 'production'
  OAUTH_CLIENT_ID = 'client_id'
  OAUTH_CLIENT_SECRET = 'secret'
  OAUTH_ALLOWED_SCOPES = ('catalog')
  ENABLED = TRUE;
```

**Supported Operations:**
- INSERT, UPDATE, DELETE, MERGE
- CREATE TABLE (in catalog-linked databases)
- Schema evolution

**Limitations:**
- Multi-statement transactions not supported
- CREATE TABLE AS SELECT not supported for Glue
- Server-side encryption limitations on Azure

## Discovery Patterns

### Pattern 1: Snowflake as Primary Catalog

```
Snowflake (CATALOG='SNOWFLAKE')
     │
     ├── Spark (via Iceberg REST endpoint)
     ├── Trino (via Iceberg REST endpoint)
     └── Other engines
```

### Pattern 2: External Catalog as Primary

```
AWS Glue / Polaris / Open Catalog
     │
     ├── Snowflake (CATALOG='glue_integration')
     ├── Spark (native Glue support)
     ├── Trino (native Glue support)
     └── EMR, Athena, etc.
```

## Best Practices

1. **Choose catalog based on primary workload**: Snowflake-heavy → Snowflake catalog
2. **Use catalog-linked databases** for multi-engine write scenarios
3. **Consider access control model**: Different catalogs have different RBAC
4. **Plan for metadata sync**: External changes may need refresh in Snowflake
