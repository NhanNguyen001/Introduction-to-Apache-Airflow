# Snowflake Production Interview Questions (2024-2025)

> Comprehensive guide covering beginner to senior-level questions for Data Engineer interviews.

---

## Table of Contents

1. [Core Concepts](#core-concepts-beginner)
2. [Architecture](#architecture-intermediate)
3. [Virtual Warehouses](#virtual-warehouses-intermediate-to-advanced)
4. [Storage and Micro-Partitions](#storage-and-micro-partitions-intermediate)
5. [Clustering and Performance](#clustering-and-performance-advanced)
6. [Data Loading](#data-loading-intermediate)
7. [Time Travel and Cloning](#time-travel-and-cloning-intermediate)
8. [Streams and Tasks (CDC)](#streams-and-tasks-cdc-advanced)
9. [Data Sharing](#data-sharing-intermediate)
10. [Security and Governance](#security-and-governance-advanced)
11. [Cost Optimization](#cost-optimization-advanced)
12. [Snowflake 2024-2025 New Features](#snowflake-2024-2025-new-features)
13. [Scenario-Based Questions](#scenario-based-questions-senior-level)
14. [SQL Performance Questions](#sql-performance-questions)

---

## Core Concepts (Beginner)

### 1. What is Snowflake and what makes it unique?

**Expected Answer:**

Snowflake is a cloud-native data platform built for the cloud from the ground up. Key differentiators:

| Feature | Description |
|---------|-------------|
| **Multi-cloud** | Runs on AWS, Azure, and GCP |
| **Separation of compute and storage** | Scale independently |
| **Zero management** | No infrastructure to manage |
| **Near-unlimited concurrency** | Via multi-cluster warehouses |
| **Native semi-structured support** | VARIANT data type for JSON, Avro, Parquet |
| **Secure data sharing** | Share data without copying |

---

### 2. Explain Snowflake's three-layer architecture.

**Expected Answer:**

```
┌─────────────────────────────────────────────────────────┐
│                   Cloud Services Layer                   │
│    (Authentication, Metadata, Query Optimization,        │
│     Access Control, Infrastructure Management)           │
├─────────────────────────────────────────────────────────┤
│                   Compute Layer                          │
│         (Virtual Warehouses - Independent MPP            │
│          Clusters for Query Processing)                  │
├─────────────────────────────────────────────────────────┤
│                   Storage Layer                          │
│     (Centralized, Compressed, Columnar Storage           │
│      in Cloud Object Storage - S3/Azure Blob/GCS)        │
└─────────────────────────────────────────────────────────┘
```

| Layer | Responsibility |
|-------|----------------|
| **Cloud Services** | Query parsing, optimization, metadata management, authentication, access control |
| **Compute (Virtual Warehouses)** | Execute queries using MPP (Massively Parallel Processing) |
| **Storage** | Store data in compressed, columnar format in micro-partitions |

---

### 3. What is the difference between Snowflake editions?

**Expected Answer:**

| Edition | Key Features |
|---------|--------------|
| **Standard** | Full SQL support, basic security, Time Travel (1 day) |
| **Enterprise** | Multi-cluster warehouses, 90-day Time Travel, materialized views, column-level security |
| **Business Critical** | HIPAA/PCI compliance, data encryption everywhere, failover/failback |
| **Virtual Private Snowflake (VPS)** | Dedicated infrastructure, highest security isolation |

---

### 4. What are the key objects in Snowflake's hierarchy?

**Expected Answer:**

```
Account
├── Database
│   ├── Schema
│   │   ├── Table
│   │   ├── View
│   │   ├── Stage
│   │   ├── Pipe
│   │   ├── Stream
│   │   ├── Task
│   │   ├── Procedure
│   │   └── Function
│   └── ...
├── Warehouse
├── User
├── Role
└── ...
```

---

### 5. What data types does Snowflake support for semi-structured data?

**Expected Answer:**

| Data Type | Description | Max Size |
|-----------|-------------|----------|
| **VARIANT** | Universal type for any valid JSON | 16 MB |
| **OBJECT** | Collection of key-value pairs | 16 MB |
| **ARRAY** | Ordered list of values | 16 MB |

```sql
-- Example: Querying semi-structured data
SELECT
    raw_json:customer.name::STRING AS customer_name,
    raw_json:items[0].price::NUMBER AS first_item_price
FROM orders;

-- Flatten nested arrays
SELECT
    value:product_id::STRING AS product_id,
    value:quantity::INT AS quantity
FROM orders, LATERAL FLATTEN(input => raw_json:items);
```

---

## Architecture (Intermediate)

### 6. How does Snowflake handle concurrency?

**Expected Answer:**

Snowflake handles concurrency through:

1. **Separate Compute Resources**: Each virtual warehouse is independent
2. **Multi-Cluster Warehouses** (Enterprise+): Auto-scale clusters based on load
3. **Result Caching**: Identical queries return cached results instantly
4. **No Resource Contention**: Queries don't compete for the same resources

```sql
-- Create multi-cluster warehouse
CREATE WAREHOUSE analytics_wh
    WAREHOUSE_SIZE = 'MEDIUM'
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 4
    SCALING_POLICY = 'STANDARD'  -- or 'ECONOMY'
    AUTO_SUSPEND = 60
    AUTO_RESUME = TRUE;
```

---

### 7. Explain Snowflake's caching layers.

**Expected Answer:**

| Cache Type | Location | Duration | Use Case |
|------------|----------|----------|----------|
| **Result Cache** | Cloud Services | 24 hours | Identical queries return instantly |
| **Local Disk Cache** | Warehouse SSD | Until warehouse suspended | Repeated table scans |
| **Remote Disk Cache** | Cloud Storage | Persistent | Raw micro-partition data |

**Key Points:**
- Result cache is FREE (no compute credits)
- Local cache is cleared when warehouse suspends
- Cache is per-warehouse (not shared)

```sql
-- Check if query used cache
SELECT * FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
WHERE QUERY_ID = 'your_query_id';
-- Look at BYTES_SCANNED (0 = result cache hit)
```

---

### 8. What is the Cloud Services layer responsible for?

**Expected Answer:**

The Cloud Services layer handles:

1. **Authentication & Access Control**: User login, RBAC
2. **Infrastructure Management**: Warehouse provisioning
3. **Metadata Management**: Object definitions, statistics
4. **Query Parsing & Optimization**: SQL compilation, execution plans
5. **Transaction Management**: ACID compliance

**Billing Note:** Cloud Services are free up to 10% of daily warehouse compute. Beyond that, you're charged.

---

## Virtual Warehouses (Intermediate to Advanced)

### 9. What are virtual warehouses and how do they work?

**Expected Answer:**

Virtual warehouses are clusters of compute resources that execute queries. Key characteristics:

| Property | Description |
|----------|-------------|
| **Independence** | Each warehouse is isolated |
| **Elasticity** | Can resize on-the-fly |
| **Per-second billing** | Minimum 60-second charge |
| **Auto-suspend/resume** | Pause when idle, resume on query |

```sql
-- Create warehouse
CREATE WAREHOUSE etl_wh
    WAREHOUSE_SIZE = 'LARGE'
    AUTO_SUSPEND = 60          -- seconds
    AUTO_RESUME = TRUE
    INITIALLY_SUSPENDED = TRUE;

-- Resize on the fly
ALTER WAREHOUSE etl_wh SET WAREHOUSE_SIZE = 'XLARGE';
```

---

### 10. How do you choose the right warehouse size?

**Expected Answer:**

| Size | Credits/Hour | Use Case |
|------|--------------|----------|
| X-Small | 1 | Simple queries, testing |
| Small | 2 | Light workloads |
| Medium | 4 | Moderate analytics |
| Large | 8 | Complex queries |
| X-Large | 16 | Heavy ETL, large scans |
| 2X-Large+ | 32+ | Very large workloads |

**Best Practices:**

1. **Start small, scale up**: Begin with X-Small, monitor performance
2. **Check for spillage**: Local/remote disk spilling indicates need for larger warehouse
3. **Separate workloads**: Different warehouses for ETL vs. BI vs. ad-hoc
4. **Monitor queue time**: High queue time = need multi-cluster or larger size

```sql
-- Check for spillage (indicates warehouse too small)
SELECT
    QUERY_ID,
    BYTES_SPILLED_TO_LOCAL_STORAGE,
    BYTES_SPILLED_TO_REMOTE_STORAGE
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
WHERE BYTES_SPILLED_TO_LOCAL_STORAGE > 0
   OR BYTES_SPILLED_TO_REMOTE_STORAGE > 0;
```

---

### 11. What is the Query Acceleration Service (QAS)?

**Expected Answer:**

QAS automatically offloads portions of eligible queries to shared compute resources:

**Benefits:**
- Reduces impact of outlier queries
- No warehouse resizing needed
- Pay only for acceleration used

```sql
-- Enable QAS
ALTER WAREHOUSE my_wh SET
    ENABLE_QUERY_ACCELERATION = TRUE
    QUERY_ACCELERATION_MAX_SCALE_FACTOR = 8;  -- 0-100

-- Check eligible queries
SELECT * FROM TABLE(INFORMATION_SCHEMA.QUERY_ACCELERATION_ELIGIBLE());
```

**Best For:**
- Large table scans
- Queries with selective filters
- Queries that would otherwise spill to disk

---

### 12. Explain multi-cluster warehouse scaling policies.

**Expected Answer:**

| Policy | Behavior | Best For |
|--------|----------|----------|
| **STANDARD** | Adds clusters quickly, removes after queries drain | Unpredictable workloads, fast response needed |
| **ECONOMY** | Waits longer before adding/removing clusters | Predictable workloads, cost-sensitive |

```sql
-- Standard scaling (prioritize performance)
ALTER WAREHOUSE bi_wh SET
    SCALING_POLICY = 'STANDARD'
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 10;

-- Economy scaling (prioritize cost)
ALTER WAREHOUSE batch_wh SET
    SCALING_POLICY = 'ECONOMY'
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 4;
```

---

## Storage and Micro-Partitions (Intermediate)

### 13. What are micro-partitions in Snowflake?

**Expected Answer:**

Micro-partitions are the fundamental storage units in Snowflake:

| Property | Value |
|----------|-------|
| **Size** | 50-500 MB uncompressed (typically ~16 MB compressed) |
| **Format** | Columnar, compressed |
| **Immutable** | Never modified, only created/deleted |
| **Auto-managed** | Snowflake handles partitioning automatically |

**Metadata stored per micro-partition:**
- Range of values for each column (min/max)
- Number of distinct values
- NULL count
- Other statistics for query optimization

---

### 14. How does partition pruning work in Snowflake?

**Expected Answer:**

Partition pruning eliminates micro-partitions that cannot contain relevant data:

```sql
-- Example: Query on date column
SELECT * FROM sales WHERE sale_date = '2024-01-15';

-- Snowflake checks metadata:
-- 1. Which micro-partitions have min_date <= '2024-01-15' AND max_date >= '2024-01-15'?
-- 2. Only scan those partitions
-- 3. Skip all others
```

**Real Impact:**
> A query on a 4.7TB table with 72 billion rows scanned 300,112 micro-partitions in 21 minutes. With proper pruning, the same query scanned just 4 micro-partitions in under 2 seconds - 741x faster!

```sql
-- Check pruning efficiency
SELECT
    PARTITIONS_SCANNED,
    PARTITIONS_TOTAL,
    ROUND(PARTITIONS_SCANNED / PARTITIONS_TOTAL * 100, 2) AS pct_scanned
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
WHERE QUERY_ID = 'your_query_id';
```

---

### 15. What is the difference between transient and permanent tables?

**Expected Answer:**

| Feature | Permanent Table | Transient Table | Temporary Table |
|---------|-----------------|-----------------|-----------------|
| **Time Travel** | Up to 90 days | 0-1 day | 0-1 day |
| **Fail-safe** | 7 days | None | None |
| **Visibility** | All users with access | All users with access | Session only |
| **Storage Cost** | Higher | Lower | Lowest |
| **Use Case** | Production data | Staging, intermediate | Session-specific |

```sql
-- Create transient table (no fail-safe, lower cost)
CREATE TRANSIENT TABLE staging_orders (
    order_id INT,
    amount DECIMAL(10,2)
);

-- Create temporary table (session-scoped)
CREATE TEMPORARY TABLE temp_results AS
SELECT * FROM orders WHERE status = 'pending';
```

---

## Clustering and Performance (Advanced)

### 16. What are clustering keys and when should you use them?

**Expected Answer:**

Clustering keys define how data is organized within micro-partitions:

**When to Use:**
- Tables > 1 TB (or > 1000 micro-partitions)
- Frequently filtered columns in WHERE clauses
- Columns used in JOIN conditions

**When NOT to Use:**
- Small tables (< 1 TB)
- Frequently updated tables (high re-clustering cost)
- High-cardinality columns without filtering patterns

```sql
-- Add clustering key
ALTER TABLE sales CLUSTER BY (sale_date, region);

-- Check clustering depth (lower is better)
SELECT SYSTEM$CLUSTERING_INFORMATION('sales', '(sale_date, region)');

-- Monitor clustering
SELECT * FROM TABLE(INFORMATION_SCHEMA.AUTOMATIC_CLUSTERING_HISTORY());
```

---

### 17. How do you choose columns for clustering keys?

**Expected Answer:**

**Best Practices:**

1. **Prioritize WHERE clause columns**: Most frequently filtered
2. **Consider cardinality**:
   - Date columns: Excellent (natural ordering)
   - Status columns: Good if selective
   - UUID: Poor choice (too random)
3. **Limit to 3-4 columns maximum**
4. **Order matters**: Most selective first
5. **For VARCHAR**: Only first 5 bytes considered

```sql
-- Good clustering key choices
ALTER TABLE events CLUSTER BY (event_date, event_type);
ALTER TABLE orders CLUSTER BY (order_date, customer_id);

-- Bad choice (high cardinality, random)
ALTER TABLE events CLUSTER BY (event_uuid);  -- Don't do this!
```

---

### 18. How do you analyze and optimize query performance?

**Expected Answer:**

**Step 1: Use Query Profile**

```sql
-- Get query ID from history
SELECT QUERY_ID, QUERY_TEXT, TOTAL_ELAPSED_TIME
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
ORDER BY START_TIME DESC
LIMIT 10;
```

**Step 2: Check Key Metrics**

| Metric | Good | Bad |
|--------|------|-----|
| Partitions Scanned vs Total | < 10% | > 50% |
| Bytes Spilled to Local | 0 | > 0 |
| Bytes Spilled to Remote | 0 | > 0 |
| Queue Time | < 1s | > 10s |

**Step 3: Common Optimizations**

```sql
-- 1. Add clustering for frequent filters
ALTER TABLE large_table CLUSTER BY (frequently_filtered_column);

-- 2. Use appropriate data types
-- Bad: VARCHAR for dates
-- Good: DATE or TIMESTAMP

-- 3. Avoid SELECT *
SELECT col1, col2 FROM table;  -- Not SELECT *

-- 4. Push filters down
SELECT * FROM table WHERE date_col = '2024-01-01'  -- Filter early
```

---

### 19. What is Search Optimization Service?

**Expected Answer:**

Search Optimization Service improves point lookup queries on large tables:

**Best For:**
- Equality predicates (`=`)
- IN clauses
- VARIANT/OBJECT field access
- Substring/regex searches

```sql
-- Enable search optimization
ALTER TABLE customers ADD SEARCH OPTIMIZATION;

-- Enable for specific columns
ALTER TABLE customers ADD SEARCH OPTIMIZATION
    ON EQUALITY(customer_id), SUBSTRING(email);

-- Check search optimization
SELECT * FROM TABLE(INFORMATION_SCHEMA.SEARCH_OPTIMIZATION_HISTORY());
```

**Cost:** 2x credit multiplier (serverless compute)

---

## Data Loading (Intermediate)

### 20. Compare COPY INTO, Snowpipe, and Snowpipe Streaming.

**Expected Answer:**

| Method | Latency | Use Case | Billing |
|--------|---------|----------|---------|
| **COPY INTO** | Manual/Batch | Bulk loads, scheduled ETL | Warehouse credits |
| **Snowpipe** | 1-2 minutes | Continuous file-based loading | Serverless credits |
| **Snowpipe Streaming** | Seconds | Real-time row-level inserts | Serverless credits |

```sql
-- COPY INTO (batch)
COPY INTO my_table
FROM @my_stage/data/
FILE_FORMAT = (TYPE = 'PARQUET')
PATTERN = '.*[.]parquet';

-- Create Snowpipe (continuous)
CREATE PIPE my_pipe AUTO_INGEST = TRUE AS
COPY INTO my_table
FROM @my_stage
FILE_FORMAT = (TYPE = 'JSON');
```

---

### 21. What are best practices for data loading?

**Expected Answer:**

**File Preparation:**

| Recommendation | Details |
|----------------|---------|
| **File Size** | 100-250 MB compressed |
| **Format** | Parquet > CSV (columnar, compressed) |
| **Compression** | GZIP, SNAPPY, or ZSTD |
| **Partitioning** | Organize by date/logical partition |

**Loading Best Practices:**

```sql
-- 1. Use staging tables for transformations
COPY INTO staging_table FROM @stage;
INSERT INTO production_table SELECT transformed_cols FROM staging_table;

-- 2. Use VALIDATION_MODE for testing
COPY INTO my_table
FROM @my_stage
VALIDATION_MODE = RETURN_ERRORS;

-- 3. Handle errors gracefully
COPY INTO my_table
FROM @my_stage
ON_ERROR = 'CONTINUE'  -- or SKIP_FILE, ABORT_STATEMENT
FORCE = FALSE;         -- Don't reload same files
```

---

### 22. What are stages and how are they used?

**Expected Answer:**

Stages are locations where data files are stored for loading/unloading:

| Stage Type | Description | Example |
|------------|-------------|---------|
| **User Stage** | Each user has one (`@~`) | `PUT file:///tmp/data.csv @~` |
| **Table Stage** | Each table has one (`@%table`) | `PUT file:///data.csv @%my_table` |
| **Named Internal** | Created explicitly | `CREATE STAGE my_stage;` |
| **Named External** | Points to cloud storage | `CREATE STAGE s3_stage URL='s3://bucket/'` |

```sql
-- Create external stage
CREATE STAGE my_s3_stage
    URL = 's3://my-bucket/data/'
    CREDENTIALS = (AWS_KEY_ID = '...' AWS_SECRET_KEY = '...')
    FILE_FORMAT = (TYPE = 'PARQUET');

-- List files in stage
LIST @my_s3_stage;

-- Load from stage
COPY INTO my_table FROM @my_s3_stage;
```

---

## Time Travel and Cloning (Intermediate)

### 23. Explain Time Travel in Snowflake.

**Expected Answer:**

Time Travel allows accessing historical data within a retention period:

| Edition | Max Retention |
|---------|---------------|
| Standard | 1 day |
| Enterprise+ | 90 days |

```sql
-- Query data at specific time
SELECT * FROM orders AT(TIMESTAMP => '2024-01-15 10:00:00'::TIMESTAMP);

-- Query data before a statement
SELECT * FROM orders BEFORE(STATEMENT => '8e5d0ca9-005e-44e6-b858-a8f5b37c5726');

-- Query data from X minutes ago
SELECT * FROM orders AT(OFFSET => -60*5);  -- 5 minutes ago

-- Restore dropped table
UNDROP TABLE accidentally_deleted_table;

-- Restore data to a table
CREATE TABLE orders_restored CLONE orders
    AT(TIMESTAMP => '2024-01-15 10:00:00'::TIMESTAMP);
```

---

### 24. What is Zero-Copy Cloning?

**Expected Answer:**

Zero-Copy Cloning creates instant copies without duplicating data:

**How It Works:**
- Clones reference the same micro-partitions
- No additional storage until data changes
- Copy-on-write: Changed data creates new micro-partitions

```sql
-- Clone a table (instant, free)
CREATE TABLE orders_dev CLONE orders;

-- Clone a schema
CREATE SCHEMA dev_schema CLONE prod_schema;

-- Clone a database
CREATE DATABASE dev_db CLONE prod_db;

-- Clone with Time Travel
CREATE TABLE orders_snapshot CLONE orders
    AT(TIMESTAMP => '2024-01-15 10:00:00'::TIMESTAMP);
```

**Use Cases:**
- Development/testing environments
- Point-in-time backups
- What-if analysis
- Data recovery

---

### 25. What is Fail-safe and how does it differ from Time Travel?

**Expected Answer:**

| Feature | Time Travel | Fail-safe |
|---------|-------------|-----------|
| **Duration** | 0-90 days (configurable) | 7 days (fixed) |
| **Access** | User-accessible | Snowflake support only |
| **Purpose** | Recovery, auditing | Disaster recovery |
| **Cost** | Storage charged | Storage charged |
| **Starts** | Immediately | After Time Travel ends |

**Total data protection:** Time Travel + Fail-safe

```sql
-- Set Time Travel retention
ALTER TABLE important_data SET DATA_RETENTION_TIME_IN_DAYS = 90;

-- Check current retention
SHOW TABLES LIKE 'important_data';
-- Look at "retention_time" column
```

---

## Streams and Tasks (CDC) (Advanced)

### 26. What are Streams and how do they enable CDC?

**Expected Answer:**

Streams capture row-level changes (CDC) on tables:

| Metadata Column | Description |
|-----------------|-------------|
| `METADATA$ACTION` | INSERT or DELETE |
| `METADATA$ISUPDATE` | TRUE if update (shows as DELETE + INSERT) |
| `METADATA$ROW_ID` | Unique row identifier |

```sql
-- Create stream on source table
CREATE STREAM orders_stream ON TABLE orders;

-- Query changes
SELECT * FROM orders_stream;

-- Process changes (consumes stream offset)
INSERT INTO orders_history
SELECT *, CURRENT_TIMESTAMP() AS processed_at
FROM orders_stream
WHERE METADATA$ACTION = 'INSERT';

-- Check if stream has data
SELECT SYSTEM$STREAM_HAS_DATA('orders_stream');
```

**Stream Types:**

| Type | Captures | Use Case |
|------|----------|----------|
| **Standard** | INSERTs, UPDATEs, DELETEs | Full CDC |
| **Append-only** | INSERTs only | Event logs, audit trails |

---

### 27. What are Tasks and how do you create pipelines?

**Expected Answer:**

Tasks schedule and automate SQL execution:

```sql
-- Create a task
CREATE TASK process_orders
    WAREHOUSE = etl_wh
    SCHEDULE = 'USING CRON 0 * * * * UTC'  -- Every hour
AS
    INSERT INTO orders_processed
    SELECT * FROM orders_stream
    WHERE METADATA$ACTION = 'INSERT';

-- Create task triggered by stream
CREATE TASK process_changes
    WAREHOUSE = etl_wh
    SCHEDULE = '1 MINUTE'
    WHEN SYSTEM$STREAM_HAS_DATA('orders_stream')
AS
    MERGE INTO orders_final t
    USING orders_stream s
    ON t.order_id = s.order_id
    WHEN MATCHED AND s.METADATA$ACTION = 'DELETE' THEN DELETE
    WHEN MATCHED THEN UPDATE SET t.amount = s.amount
    WHEN NOT MATCHED THEN INSERT (order_id, amount) VALUES (s.order_id, s.amount);

-- Create task graph (DAG)
CREATE TASK child_task
    WAREHOUSE = etl_wh
    AFTER parent_task
AS
    CALL post_process_procedure();

-- Resume tasks (they start suspended)
ALTER TASK process_orders RESUME;
```

---

### 28. How do you implement a complete CDC pipeline?

**Expected Answer:**

```sql
-- 1. Create source and target tables
CREATE TABLE source_orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    amount DECIMAL(10,2),
    status VARCHAR(20)
);

CREATE TABLE target_orders CLONE source_orders;

-- 2. Create stream on source
CREATE STREAM source_orders_stream ON TABLE source_orders;

-- 3. Create merge task
CREATE TASK sync_orders
    WAREHOUSE = cdc_wh
    SCHEDULE = '1 MINUTE'
    WHEN SYSTEM$STREAM_HAS_DATA('source_orders_stream')
AS
MERGE INTO target_orders t
USING (
    SELECT * FROM source_orders_stream
    WHERE METADATA$ACTION = 'INSERT'
) s
ON t.order_id = s.order_id
WHEN MATCHED AND s.METADATA$ISUPDATE THEN
    UPDATE SET
        t.customer_id = s.customer_id,
        t.amount = s.amount,
        t.status = s.status
WHEN NOT MATCHED THEN
    INSERT (order_id, customer_id, amount, status)
    VALUES (s.order_id, s.customer_id, s.amount, s.status);

-- 4. Handle deletes in separate task
CREATE TASK handle_deletes
    WAREHOUSE = cdc_wh
    AFTER sync_orders
AS
DELETE FROM target_orders t
WHERE EXISTS (
    SELECT 1 FROM source_orders_stream s
    WHERE s.order_id = t.order_id
    AND s.METADATA$ACTION = 'DELETE'
    AND NOT s.METADATA$ISUPDATE
);

-- 5. Resume tasks
ALTER TASK handle_deletes RESUME;
ALTER TASK sync_orders RESUME;
```

---

## Data Sharing (Intermediate)

### 29. How does Snowflake Data Sharing work?

**Expected Answer:**

Snowflake Data Sharing enables sharing data without copying:

**Key Concepts:**
- **Provider**: Account that shares data
- **Consumer**: Account that accesses shared data
- **Share**: Container for shared objects
- **Zero-copy**: Consumer queries provider's data directly

```sql
-- Provider: Create share
CREATE SHARE sales_share;

-- Grant access to objects
GRANT USAGE ON DATABASE sales_db TO SHARE sales_share;
GRANT USAGE ON SCHEMA sales_db.public TO SHARE sales_share;
GRANT SELECT ON TABLE sales_db.public.orders TO SHARE sales_share;

-- Add consumer accounts
ALTER SHARE sales_share ADD ACCOUNTS = consumer_account;

-- Consumer: Create database from share
CREATE DATABASE shared_sales FROM SHARE provider_account.sales_share;

-- Query shared data
SELECT * FROM shared_sales.public.orders;
```

---

### 30. What are Secure Views and when should you use them?

**Expected Answer:**

Secure Views hide the view definition and prevent data exposure through query optimization:

| Feature | Standard View | Secure View |
|---------|---------------|-------------|
| Definition visible | Yes | No |
| Query optimization | Full | Limited |
| Performance | Better | Slightly slower |
| Security | Basic | Enhanced |

```sql
-- Create secure view (for data sharing)
CREATE SECURE VIEW customer_summary AS
SELECT
    customer_id,
    customer_name,
    total_orders
FROM customers
WHERE region = CURRENT_ROLE();  -- Row-level security

-- Required for data sharing
GRANT SELECT ON SECURE VIEW customer_summary TO SHARE my_share;
```

**Use Secure Views when:**
- Sharing data externally
- Implementing row-level security
- Hiding business logic
- Preventing query plan exposure

---

## Security and Governance (Advanced)

### 31. Explain Snowflake's access control model.

**Expected Answer:**

Snowflake uses two access control frameworks:

**1. RBAC (Role-Based Access Control):**

```
ACCOUNTADMIN
    ├── SYSADMIN
    │   ├── Custom Roles
    │   └── Database Owners
    ├── SECURITYADMIN
    │   └── User/Role Management
    └── USERADMIN
        └── User Creation
```

**2. DAC (Discretionary Access Control):**
- Object owners control access to their objects

```sql
-- Create custom role hierarchy
CREATE ROLE data_analyst;
CREATE ROLE data_engineer;
CREATE ROLE data_admin;

-- Build hierarchy
GRANT ROLE data_analyst TO ROLE data_engineer;
GRANT ROLE data_engineer TO ROLE data_admin;

-- Grant privileges
GRANT USAGE ON WAREHOUSE analytics_wh TO ROLE data_analyst;
GRANT USAGE ON DATABASE analytics_db TO ROLE data_analyst;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics_db.public TO ROLE data_analyst;

-- Assign to users
GRANT ROLE data_analyst TO USER john_doe;
```

---

### 32. What are Row Access Policies?

**Expected Answer:**

Row Access Policies filter rows based on user context:

```sql
-- Create mapping table
CREATE TABLE region_access (
    role_name VARCHAR,
    allowed_region VARCHAR
);

INSERT INTO region_access VALUES
    ('ANALYST_NA', 'North America'),
    ('ANALYST_EU', 'Europe'),
    ('ADMIN', 'ALL');

-- Create row access policy
CREATE ROW ACCESS POLICY region_policy AS (region VARCHAR)
RETURNS BOOLEAN ->
    EXISTS (
        SELECT 1 FROM region_access
        WHERE role_name = CURRENT_ROLE()
        AND (allowed_region = region OR allowed_region = 'ALL')
    );

-- Apply policy to table
ALTER TABLE sales ADD ROW ACCESS POLICY region_policy ON (region);

-- Now queries automatically filter by role
SELECT * FROM sales;  -- Only sees allowed regions
```

---

### 33. What is Dynamic Data Masking?

**Expected Answer:**

Masking policies protect sensitive data by transforming column values:

```sql
-- Create masking policy
CREATE MASKING POLICY email_mask AS (val STRING)
RETURNS STRING ->
    CASE
        WHEN CURRENT_ROLE() IN ('ADMIN', 'PII_VIEWER') THEN val
        ELSE REGEXP_REPLACE(val, '.+@', '****@')
    END;

-- Create partial SSN mask
CREATE MASKING POLICY ssn_mask AS (val STRING)
RETURNS STRING ->
    CASE
        WHEN CURRENT_ROLE() IN ('HR_ADMIN') THEN val
        WHEN CURRENT_ROLE() IN ('HR_VIEWER') THEN 'XXX-XX-' || RIGHT(val, 4)
        ELSE '***-**-****'
    END;

-- Apply to columns
ALTER TABLE employees MODIFY COLUMN email
    SET MASKING POLICY email_mask;

ALTER TABLE employees MODIFY COLUMN ssn
    SET MASKING POLICY ssn_mask;
```

---

### 34. How do you implement data governance in Snowflake?

**Expected Answer:**

**Key Governance Features:**

| Feature | Purpose | Edition |
|---------|---------|---------|
| **Object Tagging** | Classify and track data | Enterprise+ |
| **Access History** | Audit data access | Enterprise+ |
| **Data Classification** | Auto-detect sensitive data | Enterprise+ |
| **Row Access Policies** | Row-level security | Enterprise+ |
| **Masking Policies** | Column-level security | Enterprise+ |

```sql
-- Create and apply tags
CREATE TAG pii_type ALLOWED_VALUES 'email', 'ssn', 'phone', 'address';
CREATE TAG data_sensitivity ALLOWED_VALUES 'public', 'internal', 'confidential', 'restricted';

ALTER TABLE customers MODIFY COLUMN email
    SET TAG pii_type = 'email', data_sensitivity = 'confidential';

-- Query tagged objects
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
WHERE TAG_NAME = 'PII_TYPE';

-- Access history for auditing
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY
WHERE QUERY_START_TIME > DATEADD(day, -7, CURRENT_TIMESTAMP());
```

---

## Cost Optimization (Advanced)

### 35. How is Snowflake pricing structured?

**Expected Answer:**

| Cost Component | How It's Charged | Typical % of Bill |
|----------------|------------------|-------------------|
| **Compute** | Credits per second (min 60s) | ~80% |
| **Storage** | $/TB/month (compressed) | ~15% |
| **Data Transfer** | Egress charges | ~5% |
| **Serverless** | Credits with multiplier | Varies |

**Credit Costs by Size:**

| Warehouse Size | Credits/Hour |
|----------------|--------------|
| X-Small | 1 |
| Small | 2 |
| Medium | 4 |
| Large | 8 |
| X-Large | 16 |
| 2X-Large | 32 |

---

### 36. What are the top strategies for cost optimization?

**Expected Answer:**

**1. Right-size Warehouses:**

```sql
-- Check for spillage (warehouse too small)
SELECT
    WAREHOUSE_NAME,
    AVG(BYTES_SPILLED_TO_LOCAL_STORAGE) AS avg_local_spill,
    AVG(BYTES_SPILLED_TO_REMOTE_STORAGE) AS avg_remote_spill
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE START_TIME > DATEADD(day, -7, CURRENT_TIMESTAMP())
GROUP BY WAREHOUSE_NAME;
```

**2. Aggressive Auto-Suspend:**

```sql
-- Set 60-second auto-suspend (not default 5 minutes!)
ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = 60;
```

**3. Monitor Idle Warehouses:**

```sql
-- Find warehouses with low utilization
SELECT
    WAREHOUSE_NAME,
    SUM(CREDITS_USED) AS total_credits,
    COUNT(DISTINCT QUERY_ID) AS query_count,
    SUM(CREDITS_USED) / NULLIF(COUNT(DISTINCT QUERY_ID), 0) AS credits_per_query
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE START_TIME > DATEADD(day, -30, CURRENT_TIMESTAMP())
GROUP BY WAREHOUSE_NAME
ORDER BY total_credits DESC;
```

**4. Use Result Caching:**

```sql
-- Ensure caching is enabled (default)
ALTER SESSION SET USE_CACHED_RESULT = TRUE;
```

**5. Optimize Clustering Costs:**

```sql
-- Monitor automatic clustering costs
SELECT
    TABLE_NAME,
    SUM(CREDITS_USED) AS clustering_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.AUTOMATIC_CLUSTERING_HISTORY
WHERE START_TIME > DATEADD(day, -30, CURRENT_TIMESTAMP())
GROUP BY TABLE_NAME
ORDER BY clustering_credits DESC;
```

---

### 37. How do you monitor and control Snowflake costs?

**Expected Answer:**

```sql
-- 1. Total credit usage by warehouse
SELECT
    WAREHOUSE_NAME,
    SUM(CREDITS_USED) AS total_credits,
    SUM(CREDITS_USED) * 3 AS estimated_cost  -- Assume $3/credit
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE START_TIME > DATEADD(month, -1, CURRENT_TIMESTAMP())
GROUP BY WAREHOUSE_NAME
ORDER BY total_credits DESC;

-- 2. Storage costs
SELECT
    DATABASE_NAME,
    AVG(AVERAGE_DATABASE_BYTES) / POWER(1024, 4) AS avg_tb,
    AVG(AVERAGE_DATABASE_BYTES) / POWER(1024, 4) * 23 AS monthly_cost  -- $23/TB
FROM SNOWFLAKE.ACCOUNT_USAGE.DATABASE_STORAGE_USAGE_HISTORY
WHERE USAGE_DATE > DATEADD(month, -1, CURRENT_DATE())
GROUP BY DATABASE_NAME
ORDER BY avg_tb DESC;

-- 3. Set up resource monitors
CREATE RESOURCE MONITOR monthly_limit
    WITH CREDIT_QUOTA = 1000
    FREQUENCY = MONTHLY
    START_TIMESTAMP = IMMEDIATELY
    TRIGGERS
        ON 75 PERCENT DO NOTIFY
        ON 90 PERCENT DO NOTIFY
        ON 100 PERCENT DO SUSPEND;

ALTER WAREHOUSE analytics_wh SET RESOURCE_MONITOR = monthly_limit;
```

---

## Snowflake 2024-2025 New Features

### 38. What are Dynamic Tables?

**Expected Answer:**

Dynamic Tables provide declarative data transformation with automatic refresh:

```sql
-- Create dynamic table (replaces streams + tasks for simple pipelines)
CREATE DYNAMIC TABLE sales_summary
    TARGET_LAG = '1 hour'  -- or DOWNSTREAM
    WAREHOUSE = transform_wh
AS
SELECT
    DATE_TRUNC('day', sale_date) AS sale_day,
    region,
    SUM(amount) AS total_sales,
    COUNT(*) AS transaction_count
FROM raw_sales
GROUP BY 1, 2;

-- Query like regular table
SELECT * FROM sales_summary WHERE sale_day = '2024-01-15';

-- Monitor refresh history
SELECT * FROM TABLE(INFORMATION_SCHEMA.DYNAMIC_TABLE_REFRESH_HISTORY());
```

**Benefits:**
- Declarative (just write SELECT)
- Automatic incremental refresh
- Simpler than streams + tasks
- Built-in dependency management

---

### 39. What is Snowflake Cortex AI?

**Expected Answer:**

Cortex AI brings ML/AI capabilities directly to Snowflake:

| Feature | Description |
|---------|-------------|
| **Cortex LLM Functions** | Access LLMs (Llama, Mistral) via SQL |
| **Cortex Analyst** | Natural language to SQL |
| **Cortex Search** | Semantic search on documents |
| **Cortex Fine-Tuning** | Customize models on your data |
| **Cortex Guard** | Filter harmful content |

```sql
-- Sentiment analysis
SELECT
    review_text,
    SNOWFLAKE.CORTEX.SENTIMENT(review_text) AS sentiment_score
FROM product_reviews;

-- Text summarization
SELECT
    SNOWFLAKE.CORTEX.SUMMARIZE(long_document) AS summary
FROM documents;

-- Translation
SELECT
    SNOWFLAKE.CORTEX.TRANSLATE(text, 'en', 'es') AS spanish_text
FROM content;

-- Complete (LLM generation)
SELECT
    SNOWFLAKE.CORTEX.COMPLETE(
        'mistral-large',
        'Explain the benefits of data warehousing in 3 bullet points'
    ) AS response;
```

---

### 40. What are Iceberg Tables in Snowflake?

**Expected Answer:**

Iceberg Tables use Apache Iceberg open format while leveraging Snowflake's engine:

**Benefits:**
- Open format (Parquet files + Iceberg metadata)
- Query from multiple engines (Spark, Trino, etc.)
- Schema evolution without rewrites
- Time travel at file level
- Interoperability with data lake tools

```sql
-- Create Iceberg table
CREATE ICEBERG TABLE sales_iceberg (
    sale_id INT,
    sale_date DATE,
    amount DECIMAL(10,2)
)
CATALOG = 'SNOWFLAKE'
EXTERNAL_VOLUME = 'my_s3_volume'
BASE_LOCATION = 'sales/';

-- Convert existing table to Iceberg
CREATE ICEBERG TABLE new_iceberg_table
    COPY GRANTS
    AS SELECT * FROM existing_table;

-- Query like regular table
SELECT * FROM sales_iceberg WHERE sale_date > '2024-01-01';
```

---

### 41. What is Snowflake Intelligence (2025)?

**Expected Answer:**

Snowflake Intelligence is an agentic AI framework (public preview August 2025):

**Capabilities:**
- Conversational data analysis
- Processes structured, semi-structured, and unstructured data
- Autonomous task execution
- Integration with Cortex Agents API

**Key Components:**
- Natural language queries
- Automated insights generation
- Cross-data-type reasoning
- Action recommendations

---

## Scenario-Based Questions (Senior Level)

### 42. Design a real-time analytics pipeline for e-commerce data.

**Expected Answer:**

```sql
-- 1. Create raw landing table
CREATE TABLE raw_events (
    event_id VARCHAR,
    event_timestamp TIMESTAMP,
    event_type VARCHAR,
    user_id VARCHAR,
    product_id VARCHAR,
    event_data VARIANT,
    _loaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

-- 2. Set up Snowpipe for continuous ingestion
CREATE PIPE events_pipe AUTO_INGEST = TRUE AS
COPY INTO raw_events (event_id, event_timestamp, event_type, user_id, product_id, event_data)
FROM (
    SELECT
        $1:event_id::VARCHAR,
        $1:timestamp::TIMESTAMP,
        $1:type::VARCHAR,
        $1:user_id::VARCHAR,
        $1:product_id::VARCHAR,
        $1
    FROM @events_stage
)
FILE_FORMAT = (TYPE = 'JSON');

-- 3. Create Dynamic Tables for transformations
CREATE DYNAMIC TABLE user_sessions
    TARGET_LAG = '5 minutes'
    WAREHOUSE = transform_wh
AS
SELECT
    user_id,
    DATE_TRUNC('hour', event_timestamp) AS session_hour,
    COUNT(*) AS event_count,
    COUNT(DISTINCT product_id) AS products_viewed,
    SUM(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) AS purchases
FROM raw_events
WHERE event_timestamp > DATEADD(day, -7, CURRENT_TIMESTAMP())
GROUP BY 1, 2;

-- 4. Create real-time dashboard view
CREATE DYNAMIC TABLE realtime_metrics
    TARGET_LAG = '1 minute'
    WAREHOUSE = dashboard_wh
AS
SELECT
    DATE_TRUNC('minute', event_timestamp) AS minute,
    COUNT(*) AS events_per_minute,
    COUNT(DISTINCT user_id) AS active_users,
    SUM(CASE WHEN event_type = 'purchase' THEN event_data:amount::DECIMAL END) AS revenue
FROM raw_events
WHERE event_timestamp > DATEADD(hour, -1, CURRENT_TIMESTAMP())
GROUP BY 1;
```

---

### 43. How would you migrate a large on-premise data warehouse to Snowflake?

**Expected Answer:**

**Phase 1: Assessment**
- Inventory existing objects (tables, views, procedures)
- Analyze data volumes and growth patterns
- Document dependencies and data flows
- Identify transformation requirements

**Phase 2: Schema Migration**

```sql
-- Convert DDL (example: Oracle to Snowflake)
-- Oracle:
-- CREATE TABLE orders (
--     order_id NUMBER(10),
--     order_date DATE,
--     amount NUMBER(10,2)
-- );

-- Snowflake:
CREATE TABLE orders (
    order_id INTEGER,
    order_date DATE,
    amount DECIMAL(10,2)
);
```

**Phase 3: Data Migration**

```sql
-- Option 1: Direct load from cloud storage
COPY INTO orders
FROM 's3://migration-bucket/orders/'
CREDENTIALS = (AWS_KEY_ID = '...' AWS_SECRET_KEY = '...')
FILE_FORMAT = (TYPE = 'PARQUET');

-- Option 2: Use Snowflake connector with source DB
-- (Configure via partner tools like Fivetran, Airbyte)

-- Option 3: For very large tables, use parallel loading
-- Split data by date ranges, load concurrently
```

**Phase 4: Validation**

```sql
-- Row count validation
SELECT
    'source' AS system, COUNT(*) AS row_count FROM source_table
UNION ALL
SELECT
    'snowflake', COUNT(*) FROM snowflake_table;

-- Checksum validation
SELECT MD5(LISTAGG(column1 || column2, ',')) FROM table;
```

---

### 44. Your Snowflake costs have increased 3x. How do you investigate and optimize?

**Expected Answer:**

**Step 1: Identify Cost Drivers**

```sql
-- Credit usage by warehouse
SELECT
    WAREHOUSE_NAME,
    SUM(CREDITS_USED) AS credits_this_month,
    LAG(SUM(CREDITS_USED)) OVER (ORDER BY WAREHOUSE_NAME) AS credits_last_month
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE START_TIME > DATEADD(month, -2, CURRENT_TIMESTAMP())
GROUP BY WAREHOUSE_NAME, DATE_TRUNC('month', START_TIME)
ORDER BY credits_this_month DESC;

-- Top expensive queries
SELECT
    QUERY_TEXT,
    WAREHOUSE_NAME,
    TOTAL_ELAPSED_TIME / 1000 AS seconds,
    CREDITS_USED_CLOUD_SERVICES
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE START_TIME > DATEADD(day, -7, CURRENT_TIMESTAMP())
ORDER BY TOTAL_ELAPSED_TIME DESC
LIMIT 20;
```

**Step 2: Check for Issues**

```sql
-- Warehouses running but idle
SELECT
    WAREHOUSE_NAME,
    SUM(CREDITS_USED) AS credits,
    COUNT(DISTINCT QUERY_ID) AS queries,
    SUM(CREDITS_USED) / NULLIF(COUNT(DISTINCT QUERY_ID), 0) AS credits_per_query
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY wmh
LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY qh
    ON wmh.WAREHOUSE_NAME = qh.WAREHOUSE_NAME
    AND qh.START_TIME BETWEEN wmh.START_TIME AND wmh.END_TIME
GROUP BY wmh.WAREHOUSE_NAME
HAVING credits_per_query > 0.1;  -- Flag high idle time

-- Check auto-suspend settings
SHOW WAREHOUSES;
```

**Step 3: Implement Optimizations**

```sql
-- Fix auto-suspend
ALTER WAREHOUSE expensive_wh SET AUTO_SUSPEND = 60;

-- Right-size based on queue time and spillage
ALTER WAREHOUSE oversized_wh SET WAREHOUSE_SIZE = 'MEDIUM';

-- Set up resource monitors
CREATE RESOURCE MONITOR cost_control
    WITH CREDIT_QUOTA = 5000
    TRIGGERS ON 80 PERCENT DO NOTIFY;
```

---

## SQL Performance Questions

### 45. Optimize this slow query.

**Original Query:**

```sql
SELECT
    c.customer_name,
    COUNT(o.order_id) AS order_count,
    SUM(o.amount) AS total_amount
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date >= '2024-01-01'
GROUP BY c.customer_name
ORDER BY total_amount DESC;
```

**Issues and Optimizations:**

```sql
-- Issue 1: LEFT JOIN with WHERE on right table acts as INNER JOIN
-- Issue 2: No partition pruning if order_date not clustered
-- Issue 3: Selecting customer_name requires join even if not needed

-- Optimized Version 1: If you need all customers (even without orders)
SELECT
    c.customer_name,
    COALESCE(agg.order_count, 0) AS order_count,
    COALESCE(agg.total_amount, 0) AS total_amount
FROM customers c
LEFT JOIN (
    SELECT
        customer_id,
        COUNT(*) AS order_count,
        SUM(amount) AS total_amount
    FROM orders
    WHERE order_date >= '2024-01-01'
    GROUP BY customer_id
) agg ON c.customer_id = agg.customer_id
ORDER BY total_amount DESC;

-- Optimized Version 2: If you only need customers with orders
SELECT
    c.customer_name,
    COUNT(*) AS order_count,
    SUM(o.amount) AS total_amount
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= '2024-01-01'
GROUP BY c.customer_id, c.customer_name  -- Include customer_id for better performance
ORDER BY total_amount DESC;
```

---

### 46. Write a query to find duplicate records efficiently.

**Expected Answer:**

```sql
-- Method 1: Using QUALIFY (Snowflake-specific, efficient)
SELECT *
FROM orders
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY customer_id, order_date, amount
    ORDER BY created_at DESC
) > 1;

-- Method 2: Using CTE for complex deduplication
WITH duplicates AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id, order_date, amount
            ORDER BY created_at DESC
        ) AS rn
    FROM orders
)
SELECT * FROM duplicates WHERE rn > 1;

-- Method 3: Count-based (for analysis)
SELECT
    customer_id,
    order_date,
    amount,
    COUNT(*) AS duplicate_count
FROM orders
GROUP BY 1, 2, 3
HAVING COUNT(*) > 1;

-- Delete duplicates, keep latest
DELETE FROM orders
WHERE order_id IN (
    SELECT order_id
    FROM orders
    QUALIFY ROW_NUMBER() OVER (
        PARTITION BY customer_id, order_date, amount
        ORDER BY created_at DESC
    ) > 1
);
```

---

### 47. Implement a Slowly Changing Dimension Type 2.

**Expected Answer:**

```sql
-- Target SCD2 table structure
CREATE TABLE dim_customer (
    customer_key INT AUTOINCREMENT,
    customer_id INT,
    customer_name VARCHAR,
    email VARCHAR,
    address VARCHAR,
    effective_date DATE,
    end_date DATE,
    is_current BOOLEAN,
    PRIMARY KEY (customer_key)
);

-- Merge statement for SCD2
MERGE INTO dim_customer t
USING (
    SELECT
        s.customer_id,
        s.customer_name,
        s.email,
        s.address,
        CURRENT_DATE() AS effective_date
    FROM staging_customer s
) s
ON t.customer_id = s.customer_id AND t.is_current = TRUE

-- Update existing current record (close it)
WHEN MATCHED AND (
    t.customer_name != s.customer_name OR
    t.email != s.email OR
    t.address != s.address
) THEN UPDATE SET
    t.end_date = CURRENT_DATE() - 1,
    t.is_current = FALSE

-- Insert new record (for changes and new customers)
WHEN NOT MATCHED THEN INSERT (
    customer_id, customer_name, email, address,
    effective_date, end_date, is_current
) VALUES (
    s.customer_id, s.customer_name, s.email, s.address,
    s.effective_date, '9999-12-31', TRUE
);

-- Insert new version for changed records (separate statement)
INSERT INTO dim_customer (
    customer_id, customer_name, email, address,
    effective_date, end_date, is_current
)
SELECT
    s.customer_id, s.customer_name, s.email, s.address,
    CURRENT_DATE(), '9999-12-31', TRUE
FROM staging_customer s
JOIN dim_customer t ON s.customer_id = t.customer_id
WHERE t.is_current = FALSE
  AND t.end_date = CURRENT_DATE() - 1;
```

---

## Quick Reference: Key SQL Functions

### Window Functions

```sql
-- Ranking
ROW_NUMBER() OVER (PARTITION BY col ORDER BY col2)
RANK() OVER (ORDER BY col)
DENSE_RANK() OVER (ORDER BY col)

-- Aggregation
SUM(col) OVER (PARTITION BY col2 ORDER BY col3 ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
AVG(col) OVER (PARTITION BY col2)
COUNT(*) OVER ()

-- Navigation
LAG(col, 1) OVER (ORDER BY date_col)
LEAD(col, 1) OVER (ORDER BY date_col)
FIRST_VALUE(col) OVER (PARTITION BY grp ORDER BY date_col)
LAST_VALUE(col) OVER (PARTITION BY grp ORDER BY date_col)
```

### Semi-Structured Data

```sql
-- Access JSON
column:key::TYPE
column['key']::TYPE
column[0]::TYPE  -- Array access

-- Flatten arrays
SELECT f.value:field::STRING
FROM table, LATERAL FLATTEN(input => json_column:array_field) f;

-- Parse JSON
PARSE_JSON('{"key": "value"}')
TO_JSON(variant_column)
OBJECT_CONSTRUCT('key1', val1, 'key2', val2)
ARRAY_CONSTRUCT(val1, val2, val3)
```

---

## Sources

- [DataCamp - Top 32 Snowflake Interview Questions](https://www.datacamp.com/blog/top-snowflake-interview-questions-for-all-levels)
- [InterviewBit - Top Snowflake Interview Questions 2025](https://www.interviewbit.com/snowflake-interview-questions/)
- [MindMajix - Top 50 Snowflake Interview Questions](https://mindmajix.com/snowflake-interview-questions)
- [Snowflake Documentation - Virtual Warehouses](https://docs.snowflake.com/en/user-guide/warehouses)
- [Snowflake Documentation - Micro-partitions & Clustering](https://docs.snowflake.com/en/user-guide/tables-clustering-micropartitions)
- [Snowflake Documentation - Streams](https://docs.snowflake.com/en/user-guide/streams-intro)
- [Snowflake Documentation - Row Access Policies](https://docs.snowflake.com/en/user-guide/security-row-intro)
- [Snowflake Summit 2024 - New Features](https://medium.com/snowflake/snowflake-summit-2024-summary-of-new-features-announced-512fc79b20ed)
- [Snowflake Build 2025 - New Features](https://medium.com/snowflake/snowflake-build-2025-summary-of-new-features-bfb5317fb7ab)
- [Snowflake Cost Optimization Guide](https://select.dev/posts/snowflake-cost-optimization)
- [ChaosGenius - Snowflake Clustering 101](https://www.chaosgenius.io/blog/snowflake-clustering/)
- [ChaosGenius - Snowflake Query Optimization](https://www.chaosgenius.io/blog/snowflake-query-tuning-part1/)

---

*Last updated: November 2025*
