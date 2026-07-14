# Oracle Database Table Partitioning — Advanced Guide

> A complete, practical reference for database architects, senior developers, and Laravel developers working with large-scale Oracle databases.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Why Table Partitioning Matters](#2-why-table-partitioning-matters)
3. [How Partitioning Improves Performance](#3-how-partitioning-improves-performance)
4. [Key Concepts and Differences](#4-key-concepts-and-differences)
5. [Oracle Partitioning Strategies](#5-oracle-partitioning-strategies)
6. [Advanced Working Example — E-Commerce Order Management System](#6-advanced-working-example--e-commerce-order-management-system)
   - 6.1 [Table Design](#61-table-design)
   - 6.2 [Partitioned Tables](#62-partitioned-tables)
   - 6.3 [Indexes](#63-indexes)
   - 6.4 [Sample Data](#64-sample-data)
   - 6.5 [Analytical Queries](#65-analytical-queries)
7. [Inspecting Partitions via Data Dictionary](#7-inspecting-partitions-via-data-dictionary)
8. [Partition Maintenance Operations](#8-partition-maintenance-operations)
9. [Performance Tuning with EXPLAIN PLAN](#9-performance-tuning-with-explain-plan)
10. [Laravel Developer Integration Guide](#10-laravel-developer-integration-guide)
11. [Practical Real-Life Use Cases](#11-practical-real-life-use-cases)
12. [Comparison Tables](#12-comparison-tables)
13. [Common Mistakes and How to Avoid Them](#13-common-mistakes-and-how-to-avoid-them)
14. [Best Practices](#14-best-practices)
15. [Conclusion](#15-conclusion)

---

## 1. Introduction

Oracle Database **table partitioning** is the physical division of a large table into smaller, independent storage segments called **partitions**, while the table remains logically unified. From an application perspective, queries and DML statements (INSERT, UPDATE, DELETE) interact with a single table — Oracle transparently routes operations to the correct partition(s).

Partitioning is an **Oracle Partitioning** licensed feature (available in Enterprise Edition), designed to tackle the performance, manageability, and availability challenges that arise when tables grow to hundreds of millions or billions of rows.

A partitioned table stores each partition in its own **segment**, potentially on different tablespaces, allowing fine-grained control over storage placement, backup strategy, and I/O distribution.

---

## 2. Why Table Partitioning Matters

In large enterprise systems — banking, e-commerce, telecom, healthcare, government — a single business table can accumulate:

- **Hundreds of millions of rows** within months
- **Terabytes of data** spread over years of history
- **Concurrent read/write pressure** from thousands of users and batch processes simultaneously

Without partitioning, these systems suffer from:

- Full table scans on every query because the optimizer cannot isolate relevant rows
- Long-running maintenance windows for archiving or purging old data
- Contention on indexes that must span the entire table
- Slow bulk loads that lock the whole table
- Report queries running for hours instead of minutes

Partitioning solves these problems by allowing Oracle to work on a **targeted subset** of the data rather than the whole table.

---

## 3. How Partitioning Improves Performance

### Query Performance

When a `WHERE` clause filters on the partition key, Oracle's optimizer performs **partition pruning** — it reads only the partitions that can contain relevant rows and completely skips the rest. For a table partitioned monthly over 5 years (60 partitions), a query for a single month reads 1/60th of the data.

### Data Maintenance

Archiving or purging old records in a non-partitioned table requires row-by-row DELETE, which generates massive redo logs and holds locks. With partitioning, you can `TRUNCATE PARTITION` or `DROP PARTITION` instantly — these are metadata operations with minimal redo generation.

### Data Archiving

Old partitions (e.g., orders from 3 years ago) can be moved to a cheaper tablespace, exported, or exchanged with an external table — all without touching current operational data.

### Bulk Loading

A staging table can be loaded in parallel and then attached to the main partitioned table in seconds via `EXCHANGE PARTITION`, avoiding slow row-by-row inserts into a live table.

### Index Management

Local indexes are physically co-located with their partition. Rebuilding an index for one partition (e.g., after bulk load) is fast and does not affect other partitions. This avoids the massive full-index rebuild required on a non-partitioned table.

### Large Report Generation

Report queries that aggregate historical data by month, region, or product line benefit enormously. Oracle can execute partition-level parallel queries, distributing work across CPUs — each thread operates on a separate partition.

---

## 4. Key Concepts and Differences

### Partitioned Table vs Normal Table

| Aspect | Normal Table | Partitioned Table |
|---|---|---|
| Physical storage | Single segment | Multiple segments (one per partition) |
| Query scope | Always full table | Pruned to relevant partitions |
| Maintenance | Row-by-row operations | Partition-level operations |
| Availability | All-or-nothing | Per-partition operations possible |
| Complexity | Simple | Requires partitioning strategy design |

### Partition Key vs Primary Key

- **Partition key**: The column(s) Oracle uses to determine which partition a row belongs to. Does not need to be unique. Example: `ORDER_DATE`.
- **Primary key**: The unique row identifier. Example: `ORDER_ID`. The primary key may or may not include the partition key.

Including the partition key in the primary key (e.g., `PRIMARY KEY (ORDER_ID, ORDER_DATE)`) enables **local indexes**, which are the most maintainable option.

### Local Index vs Global Index

- **Local index**: Each partition of the table has its own corresponding index partition. The index is co-located with the data. Rebuilding a partition only requires rebuilding that index partition.
- **Global index**: A single index spanning all partitions. More flexible for unique constraints not containing the partition key, but when a partition is dropped or truncated, all global index partitions are marked `UNUSABLE` unless you use `UPDATE INDEXES`.

### Partition Pruning vs Full Table Scan

- **Full table scan**: Oracle reads every block of every partition regardless of the query filter. Unavoidable on non-partitioned tables.
- **Partition pruning**: Oracle evaluates the `WHERE` clause at parse time (static pruning) or at runtime (dynamic pruning), identifies which partitions could contain matching rows, and reads only those partitions. The rest are skipped entirely.

### Partitioning vs Indexing

Partitioning and indexing solve different problems and work best together:

| | Partitioning | Indexing |
|---|---|---|
| Best for | Range/bulk queries, maintenance | Point lookups, selective queries |
| Granularity | Partition level | Row level |
| Maintenance cost | Low (partition operations) | Higher on large tables |
| Parallel execution | Natural (per partition) | Limited |
| Replaces the other? | No | No |

---

## 5. Oracle Partitioning Strategies

### Range Partitioning

Rows are assigned to partitions based on whether the partition key value falls within a defined range.

**Best for**: Time-series data — orders, transactions, logs, audit records.

```sql
PARTITION BY RANGE (ORDER_DATE) (
    PARTITION p_2023_01 VALUES LESS THAN (DATE '2023-02-01'),
    PARTITION p_2023_02 VALUES LESS THAN (DATE '2023-03-01'),
    ...
    PARTITION p_max    VALUES LESS THAN (MAXVALUE)
)
```

**Real-life examples**: Monthly bank statements, yearly tax records, CDR (Call Detail Records) in telecom.

---

### List Partitioning

Rows are assigned based on a discrete list of values for the partition key.

**Best for**: Categorical data with known, finite values — region, country, status, department.

```sql
PARTITION BY LIST (REGION) (
    PARTITION p_north VALUES ('NORTH', 'NORTHWEST', 'NORTHEAST'),
    PARTITION p_south VALUES ('SOUTH', 'SOUTHWEST', 'SOUTHEAST'),
    PARTITION p_east  VALUES ('EAST'),
    PARTITION p_west  VALUES ('WEST'),
    PARTITION p_other VALUES (DEFAULT)
)
```

**Real-life examples**: Regional sales databases, country-specific data isolation, order status routing.

---

### Hash Partitioning

Rows are distributed across a fixed number of partitions using an internal hash function on the partition key. The distribution is even but not predictable.

**Best for**: When there is no natural range or list key, and the goal is even I/O distribution and parallel query performance.

```sql
PARTITION BY HASH (CUSTOMER_ID)
PARTITIONS 8;
```

**Real-life examples**: Customer profile tables without a natural range key, product catalog tables.

---

### Composite Partitioning

Combines two strategies — a primary partition method and a sub-partition method within each primary partition.

**Common combinations**:
- `RANGE-LIST`: Partition by date, sub-partition by region
- `RANGE-HASH`: Partition by date, sub-partition for I/O balance
- `LIST-RANGE`: Partition by region, sub-partition by date

**Real-life examples**: Global e-commerce orders partitioned by month and then by region.

```sql
PARTITION BY RANGE (ORDER_DATE)
SUBPARTITION BY LIST (REGION)
SUBPARTITION TEMPLATE (
    SUBPARTITION sp_north VALUES ('NORTH'),
    SUBPARTITION sp_south VALUES ('SOUTH'),
    SUBPARTITION sp_east  VALUES ('EAST'),
    SUBPARTITION sp_west  VALUES ('WEST'),
    SUBPARTITION sp_other VALUES (DEFAULT)
)
(
    PARTITION p_2024_01 VALUES LESS THAN (DATE '2024-02-01'),
    PARTITION p_2024_02 VALUES LESS THAN (DATE '2024-03-01')
)
```

---

### Interval Partitioning

An extension of range partitioning where Oracle **automatically creates new partitions** when a row is inserted whose key value exceeds all existing partition bounds. You define the interval (e.g., one month), and Oracle does the rest.

**Best for**: Growing time-series tables where you do not want to manually add partitions each period.

```sql
PARTITION BY RANGE (ORDER_DATE)
INTERVAL (NUMTOYMINTERVAL(1, 'MONTH'))
(
    PARTITION p_initial VALUES LESS THAN (DATE '2023-01-01')
)
```

When a row with `ORDER_DATE = '2025-06-15'` is inserted, Oracle automatically creates the June 2025 partition.

---

### Reference Partitioning

A child table (with a foreign key to a partitioned parent) can be partitioned using the same strategy as the parent — without storing the partition key column in the child. Oracle resolves the partition via the foreign key relationship.

**Best for**: Child tables (order items, payment lines) that should be co-located with their parent rows for fast joins.

```sql
-- Parent table is partitioned by RANGE on ORDER_DATE
-- Child table uses reference partitioning via FK to parent

CREATE TABLE ORDER_ITEMS (
    ITEM_ID       NUMBER PRIMARY KEY,
    ORDER_ID      NUMBER NOT NULL,
    PRODUCT_ID    NUMBER,
    QUANTITY      NUMBER,
    UNIT_PRICE    NUMBER(12,2),
    CONSTRAINT fk_oi_order FOREIGN KEY (ORDER_ID)
        REFERENCES ORDERS (ORDER_ID)
)
PARTITION BY REFERENCE (fk_oi_order);
```

Now `ORDER_ITEMS` has the same partitions as `ORDERS`, and joins between them benefit from partition pruning on both sides simultaneously.

---

## 6. Advanced Working Example — E-Commerce Order Management System

This example models a realistic large-scale e-commerce system with millions of orders per month across multiple global regions.

---

### 6.1 Table Design

#### Customers Table

```sql
-- ============================================================
-- CUSTOMERS: Reference/dimension table, not partitioned.
-- Relatively small (~10M rows), queried by ID — indexing is
-- sufficient. No time-based growth pattern warrants partitioning.
-- ============================================================
CREATE TABLE CUSTOMERS (
    CUSTOMER_ID     NUMBER(12)      GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    FULL_NAME       VARCHAR2(200)   NOT NULL,
    EMAIL           VARCHAR2(300)   NOT NULL,
    PHONE           VARCHAR2(30),
    REGION          VARCHAR2(50)    NOT NULL,
    COUNTRY_CODE    CHAR(3)         NOT NULL,
    CREATED_DATE    DATE            DEFAULT SYSDATE NOT NULL,
    STATUS          VARCHAR2(20)    DEFAULT 'ACTIVE'
                    CONSTRAINT chk_cust_status CHECK (STATUS IN ('ACTIVE','INACTIVE','SUSPENDED')),
    CONSTRAINT uq_customers_email UNIQUE (EMAIL)
);

CREATE INDEX idx_customers_region ON CUSTOMERS (REGION);
CREATE INDEX idx_customers_status ON CUSTOMERS (STATUS, CREATED_DATE);
```

---

#### Products Table

```sql
-- ============================================================
-- PRODUCTS: Catalog table, not partitioned.
-- Moderate size, primarily lookup by PRODUCT_ID or CATEGORY.
-- ============================================================
CREATE TABLE PRODUCTS (
    PRODUCT_ID      NUMBER(12)      GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    PRODUCT_NAME    VARCHAR2(500)   NOT NULL,
    CATEGORY        VARCHAR2(100)   NOT NULL,
    UNIT_PRICE      NUMBER(12,2)    NOT NULL,
    STOCK_QTY       NUMBER(10)      DEFAULT 0,
    IS_ACTIVE       CHAR(1)         DEFAULT 'Y'
                    CONSTRAINT chk_prod_active CHECK (IS_ACTIVE IN ('Y','N')),
    CREATED_DATE    DATE            DEFAULT SYSDATE
);

CREATE INDEX idx_products_category ON PRODUCTS (CATEGORY);
```

---

### 6.2 Partitioned Tables

#### Orders Table — Composite Partitioning (Range-List)

```sql
-- ============================================================
-- ORDERS: Core transactional table — partitioned.
--
-- Strategy:
--   Primary partition: RANGE on ORDER_DATE (monthly)
--   Sub-partition:     LIST on REGION
--
-- Why composite?
--   - Queries filter by date range (common) AND region (common).
--   - A query like "all NORTH orders in Jan 2024" prunes to
--     exactly one sub-partition out of potentially hundreds.
--   - Partition maintenance (archiving Jan 2023) is clean —
--     DROP/TRUNCATE PARTITION removes all regions for that month.
--
-- Note: ORDER_DATE is included in the PRIMARY KEY so that
-- LOCAL indexes can be created (Oracle requires the partition
-- key to be part of any unique/PK constraint for local uniqueness).
-- ============================================================
CREATE TABLE ORDERS (
    ORDER_ID        NUMBER(15)      NOT NULL,
    ORDER_DATE      DATE            NOT NULL,
    CUSTOMER_ID     NUMBER(12)      NOT NULL,
    REGION          VARCHAR2(50)    NOT NULL,
    STATUS          VARCHAR2(30)    DEFAULT 'PENDING'
                    CONSTRAINT chk_ord_status
                    CHECK (STATUS IN ('PENDING','CONFIRMED','SHIPPED',
                                      'DELIVERED','CANCELLED','REFUNDED')),
    TOTAL_AMOUNT    NUMBER(15,2)    NOT NULL,
    CURRENCY_CODE   CHAR(3)         DEFAULT 'USD',
    PAYMENT_STATUS  VARCHAR2(20)    DEFAULT 'UNPAID'
                    CONSTRAINT chk_pay_status
                    CHECK (PAYMENT_STATUS IN ('UNPAID','PAID','PARTIAL','REFUNDED')),
    NOTES           VARCHAR2(4000),
    CREATED_BY      VARCHAR2(100),
    LAST_UPDATED    TIMESTAMP       DEFAULT SYSTIMESTAMP,
    --
    -- PK includes ORDER_DATE (partition key) to support LOCAL unique index
    CONSTRAINT pk_orders PRIMARY KEY (ORDER_ID, ORDER_DATE),
    CONSTRAINT fk_orders_customer FOREIGN KEY (CUSTOMER_ID)
        REFERENCES CUSTOMERS (CUSTOMER_ID)
)
-- Primary partition: monthly range on ORDER_DATE
PARTITION BY RANGE (ORDER_DATE)
-- Sub-partition: list on REGION (applied to every primary partition)
SUBPARTITION BY LIST (REGION)
SUBPARTITION TEMPLATE (
    SUBPARTITION sp_north VALUES ('NORTH', 'NORTHEAST', 'NORTHWEST'),
    SUBPARTITION sp_south VALUES ('SOUTH', 'SOUTHEAST', 'SOUTHWEST'),
    SUBPARTITION sp_east  VALUES ('EAST'),
    SUBPARTITION sp_west  VALUES ('WEST'),
    SUBPARTITION sp_intl  VALUES ('INTERNATIONAL'),
    SUBPARTITION sp_other VALUES (DEFAULT)
)
(
    -- Historical seed partitions
    PARTITION p_2022_q4 VALUES LESS THAN (DATE '2023-01-01')
        TABLESPACE tbs_archive,

    PARTITION p_2023_01 VALUES LESS THAN (DATE '2023-02-01')
        TABLESPACE tbs_hist_2023,
    PARTITION p_2023_02 VALUES LESS THAN (DATE '2023-03-01')
        TABLESPACE tbs_hist_2023,
    PARTITION p_2023_03 VALUES LESS THAN (DATE '2023-04-01')
        TABLESPACE tbs_hist_2023,
    PARTITION p_2023_04 VALUES LESS THAN (DATE '2023-05-01')
        TABLESPACE tbs_hist_2023,
    PARTITION p_2023_05 VALUES LESS THAN (DATE '2023-06-01')
        TABLESPACE tbs_hist_2023,
    PARTITION p_2023_06 VALUES LESS THAN (DATE '2023-07-01')
        TABLESPACE tbs_hist_2023,
    PARTITION p_2023_07 VALUES LESS THAN (DATE '2023-08-01')
        TABLESPACE tbs_hist_2023,
    PARTITION p_2023_08 VALUES LESS THAN (DATE '2023-09-01')
        TABLESPACE tbs_hist_2023,
    PARTITION p_2023_09 VALUES LESS THAN (DATE '2023-10-01')
        TABLESPACE tbs_hist_2023,
    PARTITION p_2023_10 VALUES LESS THAN (DATE '2023-11-01')
        TABLESPACE tbs_hist_2023,
    PARTITION p_2023_11 VALUES LESS THAN (DATE '2023-12-01')
        TABLESPACE tbs_hist_2023,
    PARTITION p_2023_12 VALUES LESS THAN (DATE '2024-01-01')
        TABLESPACE tbs_hist_2023,

    PARTITION p_2024_01 VALUES LESS THAN (DATE '2024-02-01')
        TABLESPACE tbs_current,
    PARTITION p_2024_02 VALUES LESS THAN (DATE '2024-03-01')
        TABLESPACE tbs_current,
    PARTITION p_2024_03 VALUES LESS THAN (DATE '2024-04-01')
        TABLESPACE tbs_current,
    PARTITION p_2024_04 VALUES LESS THAN (DATE '2024-05-01')
        TABLESPACE tbs_current,
    PARTITION p_2024_05 VALUES LESS THAN (DATE '2024-06-01')
        TABLESPACE tbs_current,
    PARTITION p_2024_06 VALUES LESS THAN (DATE '2024-07-01')
        TABLESPACE tbs_current,
    PARTITION p_2024_07 VALUES LESS THAN (DATE '2024-08-01')
        TABLESPACE tbs_current,
    PARTITION p_2024_08 VALUES LESS THAN (DATE '2024-09-01')
        TABLESPACE tbs_current,
    PARTITION p_2024_09 VALUES LESS THAN (DATE '2024-10-01')
        TABLESPACE tbs_current,
    PARTITION p_2024_10 VALUES LESS THAN (DATE '2024-11-01')
        TABLESPACE tbs_current,
    PARTITION p_2024_11 VALUES LESS THAN (DATE '2024-12-01')
        TABLESPACE tbs_current,
    PARTITION p_2024_12 VALUES LESS THAN (DATE '2025-01-01')
        TABLESPACE tbs_current,

    PARTITION p_2025_01 VALUES LESS THAN (DATE '2025-02-01')
        TABLESPACE tbs_current,
    PARTITION p_2025_02 VALUES LESS THAN (DATE '2025-03-01')
        TABLESPACE tbs_current,
    PARTITION p_2025_03 VALUES LESS THAN (DATE '2025-04-01')
        TABLESPACE tbs_current,
    PARTITION p_2025_04 VALUES LESS THAN (DATE '2025-05-01')
        TABLESPACE tbs_current,
    PARTITION p_2025_05 VALUES LESS THAN (DATE '2025-06-01')
        TABLESPACE tbs_current,

    -- Catch-all for future rows (should be replaced by INTERVAL partitioning
    -- in production; shown here for explicit control)
    PARTITION p_future  VALUES LESS THAN (MAXVALUE)
        TABLESPACE tbs_current
);
```

---

#### Order Items — Reference Partitioning

```sql
-- ============================================================
-- ORDER_ITEMS: Child of ORDERS — uses reference partitioning.
--
-- Oracle partitions ORDER_ITEMS using the same scheme as ORDERS,
-- resolved through the FK constraint. No partition key column
-- needed in ORDER_ITEMS itself — ORDER_DATE is not stored here.
--
-- Benefit: A join between ORDERS and ORDER_ITEMS on ORDER_ID
-- with a date filter causes Oracle to prune both tables to the
-- same partition(s), eliminating cross-partition joins.
-- ============================================================
CREATE TABLE ORDER_ITEMS (
    ITEM_ID         NUMBER(15)      GENERATED ALWAYS AS IDENTITY,
    ORDER_ID        NUMBER(15)      NOT NULL,
    ORDER_DATE      DATE            NOT NULL,   -- needed for FK to PK (ORDER_ID, ORDER_DATE)
    PRODUCT_ID      NUMBER(12)      NOT NULL,
    QUANTITY        NUMBER(10)      NOT NULL    CONSTRAINT chk_qty CHECK (QUANTITY > 0),
    UNIT_PRICE      NUMBER(12,2)    NOT NULL,
    DISCOUNT_PCT    NUMBER(5,2)     DEFAULT 0,
    LINE_TOTAL      NUMBER(15,2)    GENERATED ALWAYS AS
                    (QUANTITY * UNIT_PRICE * (1 - DISCOUNT_PCT / 100)) VIRTUAL,
    CONSTRAINT pk_order_items PRIMARY KEY (ITEM_ID, ORDER_ID, ORDER_DATE),
    CONSTRAINT fk_oi_order FOREIGN KEY (ORDER_ID, ORDER_DATE)
        REFERENCES ORDERS (ORDER_ID, ORDER_DATE),
    CONSTRAINT fk_oi_product FOREIGN KEY (PRODUCT_ID)
        REFERENCES PRODUCTS (PRODUCT_ID)
)
PARTITION BY REFERENCE (fk_oi_order);
```

---

#### Payments Table — Range Partitioning with Interval

```sql
-- ============================================================
-- PAYMENTS: Range-partitioned by PAYMENT_DATE.
-- Uses INTERVAL so Oracle auto-creates monthly partitions.
-- Separate from ORDERS because payment timing can differ from
-- order timing, and finance teams query payments independently.
-- ============================================================
CREATE TABLE PAYMENTS (
    PAYMENT_ID      NUMBER(15)      GENERATED ALWAYS AS IDENTITY,
    ORDER_ID        NUMBER(15)      NOT NULL,
    PAYMENT_DATE    DATE            NOT NULL,
    METHOD          VARCHAR2(50)    NOT NULL
                    CONSTRAINT chk_pay_method
                    CHECK (METHOD IN ('CREDIT_CARD','DEBIT_CARD','BANK_TRANSFER',
                                      'WALLET','CRYPTO','COD')),
    AMOUNT          NUMBER(15,2)    NOT NULL,
    CURRENCY_CODE   CHAR(3)         DEFAULT 'USD',
    GATEWAY_REF     VARCHAR2(200),
    STATUS          VARCHAR2(20)    DEFAULT 'PENDING'
                    CONSTRAINT chk_pmt_status
                    CHECK (STATUS IN ('PENDING','COMPLETED','FAILED',
                                      'REFUNDED','CHARGEBACK')),
    PROCESSED_AT    TIMESTAMP,
    CONSTRAINT pk_payments PRIMARY KEY (PAYMENT_ID, PAYMENT_DATE)
)
PARTITION BY RANGE (PAYMENT_DATE)
INTERVAL (NUMTOYMINTERVAL(1, 'MONTH'))
(
    -- Seed partition; interval partitions created automatically
    PARTITION p_pay_2022 VALUES LESS THAN (DATE '2023-01-01')
        TABLESPACE tbs_archive
);
```

---

#### Audit Logs — List Partitioning by Event Type

```sql
-- ============================================================
-- AUDIT_LOGS: High-volume event log — partitioned by LIST on
-- EVENT_TYPE. Different event categories have very different
-- volumes and retention policies:
--   - ORDER events: highest volume, 2-year retention
--   - AUTH events:  medium volume, 1-year retention
--   - PAYMENT events: medium volume, 7-year retention (compliance)
--   - ADMIN events:  low volume, indefinite retention
--
-- Within each list partition, data is further organized by date
-- using a local index (not a sub-partition, to keep it simple here).
-- ============================================================
CREATE TABLE AUDIT_LOGS (
    LOG_ID          NUMBER(18)      GENERATED ALWAYS AS IDENTITY,
    EVENT_TYPE      VARCHAR2(50)    NOT NULL
                    CONSTRAINT chk_evt_type
                    CHECK (EVENT_TYPE IN ('ORDER','AUTH','PAYMENT',
                                          'ADMIN','SYSTEM','DATA_CHANGE')),
    EVENT_DATE      DATE            NOT NULL,
    USER_ID         NUMBER(12),
    SESSION_ID      VARCHAR2(100),
    IP_ADDRESS      VARCHAR2(50),
    ENTITY_TYPE     VARCHAR2(100),
    ENTITY_ID       NUMBER(15),
    ACTION          VARCHAR2(100)   NOT NULL,
    OLD_VALUE       CLOB,
    NEW_VALUE       CLOB,
    STATUS          VARCHAR2(20)    DEFAULT 'SUCCESS',
    ERROR_MESSAGE   VARCHAR2(4000),
    CONSTRAINT pk_audit_logs PRIMARY KEY (LOG_ID, EVENT_TYPE)
)
PARTITION BY LIST (EVENT_TYPE) (
    PARTITION p_audit_order   VALUES ('ORDER')        TABLESPACE tbs_audit_order,
    PARTITION p_audit_auth    VALUES ('AUTH')         TABLESPACE tbs_audit_auth,
    PARTITION p_audit_payment VALUES ('PAYMENT')      TABLESPACE tbs_audit_payment,
    PARTITION p_audit_admin   VALUES ('ADMIN')        TABLESPACE tbs_audit_admin,
    PARTITION p_audit_system  VALUES ('SYSTEM')       TABLESPACE tbs_audit_system,
    PARTITION p_audit_data    VALUES ('DATA_CHANGE')  TABLESPACE tbs_audit_data
);
```

---

### 6.3 Indexes

```sql
-- ============================================================
-- LOCAL INDEXES on ORDERS
-- A local index is automatically partitioned in alignment with
-- the table. Each index partition maps to one table partition.
-- When a table partition is added/dropped, the local index
-- partition follows automatically.
-- ============================================================

-- Local index on CUSTOMER_ID: supports "all orders for customer X"
-- filtered within a date range (pruning limits how many partitions
-- the index scan touches).
CREATE INDEX idx_orders_customer_local
    ON ORDERS (CUSTOMER_ID, ORDER_DATE)
    LOCAL;

-- Local index on STATUS + ORDER_DATE: supports order status dashboards
-- by period.
CREATE INDEX idx_orders_status_local
    ON ORDERS (STATUS, ORDER_DATE)
    LOCAL;

-- Local index on REGION + ORDER_DATE: supports regional queries.
CREATE INDEX idx_orders_region_local
    ON ORDERS (REGION, ORDER_DATE)
    LOCAL;

-- ============================================================
-- GLOBAL INDEXES on ORDERS
-- A global index spans all partitions. Useful for lookups by
-- ORDER_ID alone (without knowing ORDER_DATE), e.g., when an
-- application only has ORDER_ID.
-- IMPORTANT: When a partition is dropped/truncated, global indexes
-- become UNUSABLE unless you use UPDATE INDEXES clause.
-- ============================================================

-- Global index: allows lookup by ORDER_ID alone without knowing ORDER_DATE.
CREATE UNIQUE INDEX idx_orders_id_global
    ON ORDERS (ORDER_ID)
    GLOBAL;

-- Global index on PAYMENT_STATUS for billing dashboards that don't
-- filter by date.
CREATE INDEX idx_orders_paymentstatus_global
    ON ORDERS (PAYMENT_STATUS, TOTAL_AMOUNT)
    GLOBAL;

-- ============================================================
-- LOCAL INDEXES on PAYMENTS
-- ============================================================
CREATE INDEX idx_payments_order_local
    ON PAYMENTS (ORDER_ID, PAYMENT_DATE)
    LOCAL;

CREATE INDEX idx_payments_status_local
    ON PAYMENTS (STATUS, PAYMENT_DATE)
    LOCAL;

-- ============================================================
-- LOCAL INDEXES on AUDIT_LOGS
-- ============================================================
CREATE INDEX idx_auditlogs_date_local
    ON AUDIT_LOGS (EVENT_DATE, EVENT_TYPE)
    LOCAL;

CREATE INDEX idx_auditlogs_user_local
    ON AUDIT_LOGS (USER_ID, EVENT_DATE)
    LOCAL;
```

---

### 6.4 Sample Data

```sql
-- ============================================================
-- SAMPLE: Customers
-- ============================================================
INSERT ALL
    INTO CUSTOMERS (FULL_NAME, EMAIL, PHONE, REGION, COUNTRY_CODE, STATUS)
        VALUES ('Ahmed Hassan',    'ahmed@example.com',   '+880-17', 'NORTH',         'BGD', 'ACTIVE')
    INTO CUSTOMERS (FULL_NAME, EMAIL, PHONE, REGION, COUNTRY_CODE, STATUS)
        VALUES ('Sarah Mitchell',  'sarah@example.com',   '+1-202',  'NORTH',         'USA', 'ACTIVE')
    INTO CUSTOMERS (FULL_NAME, EMAIL, PHONE, REGION, COUNTRY_CODE, STATUS)
        VALUES ('Carlos Rivera',   'carlos@example.com',  '+52-55',  'SOUTH',         'MEX', 'ACTIVE')
    INTO CUSTOMERS (FULL_NAME, EMAIL, PHONE, REGION, COUNTRY_CODE, STATUS)
        VALUES ('Li Wei',          'liwei@example.com',   '+86-10',  'EAST',          'CHN', 'ACTIVE')
    INTO CUSTOMERS (FULL_NAME, EMAIL, PHONE, REGION, COUNTRY_CODE, STATUS)
        VALUES ('Emma Schmidt',    'emma@example.com',    '+49-30',  'WEST',          'DEU', 'ACTIVE')
    INTO CUSTOMERS (FULL_NAME, EMAIL, PHONE, REGION, COUNTRY_CODE, STATUS)
        VALUES ('Ravi Patel',      'ravi@example.com',    '+91-22',  'INTERNATIONAL', 'IND', 'ACTIVE')
    INTO CUSTOMERS (FULL_NAME, EMAIL, PHONE, REGION, COUNTRY_CODE, STATUS)
        VALUES ('Fatima Al-Sayed', 'fatima@example.com',  '+971-4',  'INTERNATIONAL', 'ARE', 'ACTIVE')
    INTO CUSTOMERS (FULL_NAME, EMAIL, PHONE, REGION, COUNTRY_CODE, STATUS)
        VALUES ('James Okonkwo',   'james@example.com',   '+234-1',  'SOUTH',         'NGA', 'ACTIVE')
SELECT 1 FROM DUAL;

-- ============================================================
-- SAMPLE: Products
-- ============================================================
INSERT ALL
    INTO PRODUCTS (PRODUCT_NAME, CATEGORY, UNIT_PRICE, STOCK_QTY)
        VALUES ('Laptop Pro 15',     'ELECTRONICS', 1299.99, 500)
    INTO PRODUCTS (PRODUCT_NAME, CATEGORY, UNIT_PRICE, STOCK_QTY)
        VALUES ('Wireless Mouse',    'ACCESSORIES',   29.99, 2000)
    INTO PRODUCTS (PRODUCT_NAME, CATEGORY, UNIT_PRICE, STOCK_QTY)
        VALUES ('USB-C Hub',         'ACCESSORIES',   49.99, 1500)
    INTO PRODUCTS (PRODUCT_NAME, CATEGORY, UNIT_PRICE, STOCK_QTY)
        VALUES ('4K Monitor',        'ELECTRONICS',  399.99, 300)
    INTO PRODUCTS (PRODUCT_NAME, CATEGORY, UNIT_PRICE, STOCK_QTY)
        VALUES ('Mechanical Keyboard','ACCESSORIES',  89.99, 800)
    INTO PRODUCTS (PRODUCT_NAME, CATEGORY, UNIT_PRICE, STOCK_QTY)
        VALUES ('Running Shoes X1',  'FOOTWEAR',     119.99, 1200)
    INTO PRODUCTS (PRODUCT_NAME, CATEGORY, UNIT_PRICE, STOCK_QTY)
        VALUES ('Yoga Mat Pro',      'FITNESS',       39.99, 3000)
SELECT 1 FROM DUAL;

-- ============================================================
-- SAMPLE: Orders (routed to specific partitions by ORDER_DATE)
-- Each INSERT goes to the partition matching ORDER_DATE + REGION.
-- ============================================================
INSERT INTO ORDERS (ORDER_ID, ORDER_DATE, CUSTOMER_ID, REGION,
                    STATUS, TOTAL_AMOUNT, CURRENCY_CODE, PAYMENT_STATUS)
SELECT
    seq.n                          AS ORDER_ID,
    DATE '2024-01-01' + MOD(seq.n, 28) AS ORDER_DATE,  -- Spread across Jan 2024
    MOD(seq.n, 8) + 1              AS CUSTOMER_ID,
    CASE MOD(seq.n, 5)
        WHEN 0 THEN 'NORTH'
        WHEN 1 THEN 'SOUTH'
        WHEN 2 THEN 'EAST'
        WHEN 3 THEN 'WEST'
        ELSE        'INTERNATIONAL'
    END                            AS REGION,
    CASE MOD(seq.n, 6)
        WHEN 0 THEN 'PENDING'
        WHEN 1 THEN 'CONFIRMED'
        WHEN 2 THEN 'SHIPPED'
        WHEN 3 THEN 'DELIVERED'
        WHEN 4 THEN 'CANCELLED'
        ELSE        'REFUNDED'
    END                            AS STATUS,
    ROUND(DBMS_RANDOM.VALUE(50, 2500), 2) AS TOTAL_AMOUNT,
    'USD'                          AS CURRENCY_CODE,
    CASE WHEN MOD(seq.n, 3) = 0 THEN 'PAID' ELSE 'UNPAID' END AS PAYMENT_STATUS
FROM (
    SELECT LEVEL AS n FROM DUAL CONNECT BY LEVEL <= 500
) seq;

-- February 2024 orders
INSERT INTO ORDERS (ORDER_ID, ORDER_DATE, CUSTOMER_ID, REGION,
                    STATUS, TOTAL_AMOUNT, CURRENCY_CODE, PAYMENT_STATUS)
SELECT
    500 + seq.n,
    DATE '2024-02-01' + MOD(seq.n, 27),
    MOD(seq.n, 8) + 1,
    CASE MOD(seq.n, 5)
        WHEN 0 THEN 'NORTH'    WHEN 1 THEN 'SOUTH'
        WHEN 2 THEN 'EAST'     WHEN 3 THEN 'WEST'
        ELSE 'INTERNATIONAL'
    END,
    CASE MOD(seq.n, 4)
        WHEN 0 THEN 'CONFIRMED' WHEN 1 THEN 'SHIPPED'
        WHEN 2 THEN 'DELIVERED' ELSE 'PENDING'
    END,
    ROUND(DBMS_RANDOM.VALUE(50, 3000), 2),
    'USD',
    CASE WHEN MOD(seq.n, 2) = 0 THEN 'PAID' ELSE 'UNPAID' END
FROM (SELECT LEVEL AS n FROM DUAL CONNECT BY LEVEL <= 400) seq;

COMMIT;

-- ============================================================
-- SAMPLE: Order Items (reference partitioning — same partition
-- as the parent ORDERS row, automatically)
-- ============================================================
INSERT INTO ORDER_ITEMS (ORDER_ID, ORDER_DATE, PRODUCT_ID, QUANTITY,
                          UNIT_PRICE, DISCOUNT_PCT)
SELECT
    o.ORDER_ID,
    o.ORDER_DATE,
    MOD(o.ORDER_ID, 7) + 1  AS PRODUCT_ID,
    TRUNC(DBMS_RANDOM.VALUE(1, 5)) AS QUANTITY,
    p.UNIT_PRICE,
    CASE WHEN MOD(o.ORDER_ID, 10) = 0 THEN 10 ELSE 0 END AS DISCOUNT_PCT
FROM ORDERS o
JOIN PRODUCTS p ON p.PRODUCT_ID = MOD(o.ORDER_ID, 7) + 1
WHERE ROWNUM <= 900;

COMMIT;

-- ============================================================
-- SAMPLE: Payments
-- ============================================================
INSERT INTO PAYMENTS (ORDER_ID, PAYMENT_DATE, METHOD, AMOUNT,
                       CURRENCY_CODE, GATEWAY_REF, STATUS)
SELECT
    o.ORDER_ID,
    o.ORDER_DATE + TRUNC(DBMS_RANDOM.VALUE(0, 3)) AS PAYMENT_DATE,
    CASE MOD(o.ORDER_ID, 5)
        WHEN 0 THEN 'CREDIT_CARD'    WHEN 1 THEN 'DEBIT_CARD'
        WHEN 2 THEN 'BANK_TRANSFER'  WHEN 3 THEN 'WALLET'
        ELSE        'COD'
    END,
    o.TOTAL_AMOUNT,
    'USD',
    'GW-' || TO_CHAR(o.ORDER_ID, 'FM000000'),
    CASE WHEN o.PAYMENT_STATUS = 'PAID' THEN 'COMPLETED' ELSE 'PENDING' END
FROM ORDERS o
WHERE o.PAYMENT_STATUS = 'PAID';

COMMIT;

-- ============================================================
-- SAMPLE: Audit Logs
-- ============================================================
INSERT INTO AUDIT_LOGS (EVENT_TYPE, EVENT_DATE, USER_ID, ACTION,
                          ENTITY_TYPE, ENTITY_ID, STATUS)
SELECT
    CASE MOD(LEVEL, 6)
        WHEN 0 THEN 'ORDER'    WHEN 1 THEN 'AUTH'
        WHEN 2 THEN 'PAYMENT'  WHEN 3 THEN 'ADMIN'
        WHEN 4 THEN 'SYSTEM'   ELSE       'DATA_CHANGE'
    END,
    SYSDATE - MOD(LEVEL, 90),
    MOD(LEVEL, 8) + 1,
    CASE MOD(LEVEL, 4)
        WHEN 0 THEN 'CREATE'   WHEN 1 THEN 'UPDATE'
        WHEN 2 THEN 'DELETE'   ELSE       'VIEW'
    END,
    'ORDER',
    MOD(LEVEL, 900) + 1,
    'SUCCESS'
FROM DUAL CONNECT BY LEVEL <= 2000;

COMMIT;
```

---

### 6.5 Analytical Queries

#### Query 1 — Partition Pruning: Single Month

```sql
-- ============================================================
-- This query filters on ORDER_DATE with a literal date range.
-- Oracle performs STATIC partition pruning at parse time:
-- only partition p_2024_01 is accessed.
-- No other partition is touched — not even opened.
-- ============================================================
SELECT
    ORDER_ID,
    ORDER_DATE,
    CUSTOMER_ID,
    REGION,
    STATUS,
    TOTAL_AMOUNT
FROM ORDERS
WHERE ORDER_DATE >= DATE '2024-01-01'
  AND ORDER_DATE <  DATE '2024-02-01'
ORDER BY ORDER_DATE, ORDER_ID;
```

**Expected output (sample rows):**

| ORDER_ID | ORDER_DATE | CUSTOMER_ID | REGION | STATUS | TOTAL_AMOUNT |
|---|---|---|---|---|---|
| 1 | 2024-01-01 | 2 | SOUTH | CONFIRMED | 1,435.50 |
| 2 | 2024-01-02 | 3 | EAST | SHIPPED | 879.00 |
| 3 | 2024-01-03 | 4 | WEST | DELIVERED | 2,105.75 |
| … | … | … | … | … | … |

---

#### Query 2 — Sub-Partition Pruning: Specific Month AND Region

```sql
-- ============================================================
-- Oracle prunes to ONE sub-partition:
-- partition p_2024_01 → sub-partition sp_north
-- This is the most granular pruning possible with composite
-- partitioning — only that single sub-partition segment is read.
-- ============================================================
SELECT
    o.ORDER_ID,
    o.ORDER_DATE,
    c.FULL_NAME,
    o.STATUS,
    o.TOTAL_AMOUNT
FROM ORDERS o
JOIN CUSTOMERS c ON c.CUSTOMER_ID = o.CUSTOMER_ID
WHERE o.ORDER_DATE >= DATE '2024-01-01'
  AND o.ORDER_DATE <  DATE '2024-02-01'
  AND o.REGION = 'NORTH'
ORDER BY o.TOTAL_AMOUNT DESC;
```

---

#### Query 3 — Monthly Sales Report (Aggregation with Partition Pruning)

```sql
-- ============================================================
-- Monthly sales report for 2024.
-- Oracle accesses only the 12 partitions for 2024 (p_2024_01
-- through p_2024_12) out of all available partitions.
-- The GROUP BY and aggregation happen within those partitions
-- and can run in parallel with PARALLEL hint.
-- ============================================================
SELECT
    TO_CHAR(ORDER_DATE, 'YYYY-MM')          AS SALES_MONTH,
    REGION,
    COUNT(*)                                 AS TOTAL_ORDERS,
    SUM(TOTAL_AMOUNT)                        AS GROSS_REVENUE,
    AVG(TOTAL_AMOUNT)                        AS AVG_ORDER_VALUE,
    COUNT(CASE WHEN STATUS = 'DELIVERED'
               THEN 1 END)                   AS DELIVERED_ORDERS,
    COUNT(CASE WHEN STATUS = 'CANCELLED'
               THEN 1 END)                   AS CANCELLED_ORDERS,
    ROUND(
        COUNT(CASE WHEN STATUS = 'CANCELLED' THEN 1 END) * 100.0
        / NULLIF(COUNT(*), 0), 2
    )                                        AS CANCELLATION_RATE_PCT
FROM ORDERS
WHERE ORDER_DATE >= DATE '2024-01-01'
  AND ORDER_DATE <  DATE '2025-01-01'
GROUP BY TO_CHAR(ORDER_DATE, 'YYYY-MM'), REGION
ORDER BY SALES_MONTH, REGION;
```

**Expected output (sample):**

| SALES_MONTH | REGION | TOTAL_ORDERS | GROSS_REVENUE | AVG_ORDER_VALUE | DELIVERED_ORDERS | CANCELLED_ORDERS | CANCELLATION_RATE_PCT |
|---|---|---|---|---|---|---|---|
| 2024-01 | EAST | 24 | 31,450.00 | 1,310.42 | 10 | 4 | 16.67 |
| 2024-01 | INTERNATIONAL | 21 | 27,890.50 | 1,328.12 | 8 | 3 | 14.29 |
| 2024-01 | NORTH | 18 | 22,340.75 | 1,241.15 | 7 | 2 | 11.11 |
| … | … | … | … | … | … | … | … |

---

#### Query 4 — Top Customers by Revenue (Partitioned Data)

```sql
-- ============================================================
-- Top 10 customers by total spending in Q1 2024.
-- Pruned to partitions: p_2024_01, p_2024_02, p_2024_03.
-- ============================================================
SELECT * FROM (
    SELECT
        c.CUSTOMER_ID,
        c.FULL_NAME,
        c.REGION,
        COUNT(o.ORDER_ID)       AS TOTAL_ORDERS,
        SUM(o.TOTAL_AMOUNT)     AS TOTAL_SPENT,
        MAX(o.TOTAL_AMOUNT)     AS LARGEST_ORDER,
        RANK() OVER (ORDER BY SUM(o.TOTAL_AMOUNT) DESC) AS REVENUE_RANK
    FROM ORDERS o
    JOIN CUSTOMERS c ON c.CUSTOMER_ID = o.CUSTOMER_ID
    WHERE o.ORDER_DATE >= DATE '2024-01-01'
      AND o.ORDER_DATE <  DATE '2024-04-01'
      AND o.STATUS != 'CANCELLED'
    GROUP BY c.CUSTOMER_ID, c.FULL_NAME, c.REGION
)
WHERE REVENUE_RANK <= 10
ORDER BY REVENUE_RANK;
```

---

#### Query 5 — Revenue with Order Items Join (Reference Partition Pruning)

```sql
-- ============================================================
-- Because ORDER_ITEMS uses reference partitioning (FK to ORDERS),
-- Oracle can prune ORDER_ITEMS to the same partitions as ORDERS
-- when the join includes the partition key filter.
-- This avoids a full scan of ORDER_ITEMS (which could be 10x
-- larger than ORDERS).
-- ============================================================
SELECT
    p.CATEGORY,
    p.PRODUCT_NAME,
    SUM(oi.QUANTITY)                    AS UNITS_SOLD,
    SUM(oi.LINE_TOTAL)                  AS LINE_REVENUE,
    ROUND(AVG(oi.DISCOUNT_PCT), 2)      AS AVG_DISCOUNT_PCT,
    RANK() OVER (
        PARTITION BY p.CATEGORY
        ORDER BY SUM(oi.LINE_TOTAL) DESC
    )                                   AS RANK_IN_CATEGORY
FROM ORDER_ITEMS oi
JOIN ORDERS    o ON o.ORDER_ID   = oi.ORDER_ID
                AND o.ORDER_DATE = oi.ORDER_DATE
JOIN PRODUCTS  p ON p.PRODUCT_ID = oi.PRODUCT_ID
WHERE o.ORDER_DATE >= DATE '2024-01-01'
  AND o.ORDER_DATE <  DATE '2024-03-01'
  AND o.STATUS = 'DELIVERED'
GROUP BY p.CATEGORY, p.PRODUCT_ID, p.PRODUCT_NAME
ORDER BY p.CATEGORY, RANK_IN_CATEGORY;
```

---

#### Query 6 — Payment Summary by Method and Month

```sql
-- ============================================================
-- Payment analysis — PAYMENTS table is partitioned by PAYMENT_DATE
-- with INTERVAL partitioning. The query prunes to Q1 2024 only.
-- ============================================================
SELECT
    TO_CHAR(PAYMENT_DATE, 'YYYY-MM')    AS PAY_MONTH,
    METHOD,
    COUNT(*)                            AS TRANSACTIONS,
    SUM(AMOUNT)                         AS TOTAL_PAID,
    MIN(AMOUNT)                         AS MIN_PAYMENT,
    MAX(AMOUNT)                         AS MAX_PAYMENT,
    ROUND(AVG(AMOUNT), 2)               AS AVG_PAYMENT
FROM PAYMENTS
WHERE PAYMENT_DATE >= DATE '2024-01-01'
  AND PAYMENT_DATE <  DATE '2024-04-01'
  AND STATUS = 'COMPLETED'
GROUP BY TO_CHAR(PAYMENT_DATE, 'YYYY-MM'), METHOD
ORDER BY PAY_MONTH, TOTAL_PAID DESC;
```

---

#### Query 7 — Audit Log Query by Event Type (List Partition Pruning)

```sql
-- ============================================================
-- List partitioning on EVENT_TYPE means this query reads only
-- partition p_audit_order — not AUTH, PAYMENT, ADMIN, etc.
-- ============================================================
SELECT
    TRUNC(EVENT_DATE)       AS LOG_DATE,
    ACTION,
    COUNT(*)                AS EVENT_COUNT
FROM AUDIT_LOGS
WHERE EVENT_TYPE = 'ORDER'
  AND EVENT_DATE >= SYSDATE - 30
GROUP BY TRUNC(EVENT_DATE), ACTION
ORDER BY LOG_DATE DESC, EVENT_COUNT DESC;
```

---

## 7. Inspecting Partitions via Data Dictionary

```sql
-- ============================================================
-- USER_PART_TABLES: High-level partition info per table
-- ============================================================
SELECT
    TABLE_NAME,
    PARTITIONING_TYPE,
    SUBPARTITIONING_TYPE,
    PARTITION_COUNT,
    DEF_SUBPARTITION_COUNT,
    STATUS
FROM USER_PART_TABLES
ORDER BY TABLE_NAME;
```

**Expected output:**

| TABLE_NAME | PARTITIONING_TYPE | SUBPARTITIONING_TYPE | PARTITION_COUNT | DEF_SUBPARTITION_COUNT |
|---|---|---|---|---|
| AUDIT_LOGS | LIST | NONE | 6 | 0 |
| ORDER_ITEMS | REFERENCE | NONE | 30 | 6 |
| ORDERS | RANGE | LIST | 30 | 6 |
| PAYMENTS | RANGE | NONE | (auto) | 0 |

---

```sql
-- ============================================================
-- USER_TAB_PARTITIONS: One row per partition per table.
-- Shows segment size, row counts (after stats), tablespace.
-- ============================================================
SELECT
    TABLE_NAME,
    PARTITION_NAME,
    PARTITION_POSITION,
    HIGH_VALUE,
    NUM_ROWS,
    BLOCKS,
    AVG_ROW_LEN,
    TABLESPACE_NAME,
    LAST_ANALYZED
FROM USER_TAB_PARTITIONS
WHERE TABLE_NAME = 'ORDERS'
ORDER BY PARTITION_POSITION;
```

**Expected output (sample):**

| TABLE_NAME | PARTITION_NAME | PARTITION_POSITION | HIGH_VALUE | NUM_ROWS | TABLESPACE_NAME |
|---|---|---|---|---|---|
| ORDERS | P_2022_Q4 | 1 | DATE '2023-01-01' | 0 | TBS_ARCHIVE |
| ORDERS | P_2023_01 | 2 | DATE '2023-02-01' | 0 | TBS_HIST_2023 |
| ORDERS | P_2024_01 | 14 | DATE '2024-02-01' | 500 | TBS_CURRENT |
| ORDERS | P_2024_02 | 15 | DATE '2024-03-01' | 400 | TBS_CURRENT |
| … | … | … | … | … | … |

---

```sql
-- ============================================================
-- USER_TAB_SUBPARTITIONS: One row per sub-partition.
-- ============================================================
SELECT
    TABLE_NAME,
    PARTITION_NAME,
    SUBPARTITION_NAME,
    SUBPARTITION_POSITION,
    HIGH_VALUE,
    NUM_ROWS,
    TABLESPACE_NAME
FROM USER_TAB_SUBPARTITIONS
WHERE TABLE_NAME = 'ORDERS'
  AND PARTITION_NAME = 'P_2024_01'
ORDER BY SUBPARTITION_POSITION;
```

**Expected output:**

| TABLE_NAME | PARTITION_NAME | SUBPARTITION_NAME | SUBPARTITION_POSITION | HIGH_VALUE | NUM_ROWS |
|---|---|---|---|---|---|
| ORDERS | P_2024_01 | P_2024_01_SP_NORTH | 1 | 'NORTH', 'NORTHEAST', … | 100 |
| ORDERS | P_2024_01 | P_2024_01_SP_SOUTH | 2 | 'SOUTH', 'SOUTHEAST', … | 95 |
| ORDERS | P_2024_01 | P_2024_01_SP_EAST | 3 | 'EAST' | 88 |
| ORDERS | P_2024_01 | P_2024_01_SP_WEST | 4 | 'WEST' | 92 |
| ORDERS | P_2024_01 | P_2024_01_SP_INTL | 5 | 'INTERNATIONAL' | 80 |
| ORDERS | P_2024_01 | P_2024_01_SP_OTHER | 6 | DEFAULT | 45 |

---

```sql
-- ============================================================
-- USER_IND_PARTITIONS: Index partition info
-- ============================================================
SELECT
    INDEX_NAME,
    PARTITION_NAME,
    PARTITION_POSITION,
    STATUS,           -- USABLE or UNUSABLE
    NUM_ROWS,
    BLEVEL,
    LEAF_BLOCKS,
    LAST_ANALYZED
FROM USER_IND_PARTITIONS
WHERE INDEX_NAME = 'IDX_ORDERS_CUSTOMER_LOCAL'
ORDER BY PARTITION_POSITION;
```

---

```sql
-- ============================================================
-- Partition size on disk (requires DBA_SEGMENTS or USER_SEGMENTS)
-- ============================================================
SELECT
    SEGMENT_NAME,
    PARTITION_NAME,
    TABLESPACE_NAME,
    ROUND(BYTES / 1048576, 2) AS SIZE_MB,
    BLOCKS,
    EXTENTS
FROM USER_SEGMENTS
WHERE SEGMENT_NAME IN ('ORDERS', 'ORDER_ITEMS', 'PAYMENTS', 'AUDIT_LOGS')
ORDER BY SEGMENT_NAME, PARTITION_NAME;
```

---

## 8. Partition Maintenance Operations

### 8.1 ADD PARTITION

Add a new partition for a future month.

```sql
-- ============================================================
-- Add June 2025 partition to ORDERS.
-- The sub-partition template is applied automatically.
-- Run this as part of a monthly maintenance job.
-- ============================================================
ALTER TABLE ORDERS
ADD PARTITION p_2025_06
    VALUES LESS THAN (DATE '2025-07-01')
    TABLESPACE tbs_current;

-- Verify
SELECT PARTITION_NAME, HIGH_VALUE, TABLESPACE_NAME
FROM USER_TAB_PARTITIONS
WHERE TABLE_NAME = 'ORDERS'
ORDER BY PARTITION_POSITION DESC
FETCH FIRST 3 ROWS ONLY;
```

> **Note:** With INTERVAL partitioning (as used on PAYMENTS), you do not need `ADD PARTITION` — Oracle creates it automatically when the first row for that interval is inserted.

---

### 8.2 DROP PARTITION

Remove a partition and its data permanently. Useful for retiring data past retention period.

```sql
-- ============================================================
-- Drop the Q4 2022 archive partition from ORDERS.
-- This removes the segment and all rows in it instantly —
-- no row-by-row DELETE, no undo generation beyond metadata.
--
-- UPDATE INDEXES: rebuilds global indexes so they remain
-- USABLE. Without this clause, all global indexes on ORDERS
-- become UNUSABLE after the drop.
-- ============================================================
ALTER TABLE ORDERS
DROP PARTITION p_2022_q4
UPDATE INDEXES;

-- Verify global index is still usable
SELECT INDEX_NAME, STATUS
FROM USER_INDEXES
WHERE TABLE_NAME = 'ORDERS'
  AND PARTITIONED = 'NO';  -- Global (non-partitioned) indexes
```

---

### 8.3 TRUNCATE PARTITION

Remove all rows from a partition without dropping it. Keeps the empty partition in place.

```sql
-- ============================================================
-- Truncate a specific month's data while keeping the partition
-- structure. Useful when you need to reload data into a partition
-- (e.g., reprocessing end-of-month figures).
-- ============================================================
ALTER TABLE ORDERS
TRUNCATE PARTITION p_2023_01
UPDATE INDEXES;
```

---

### 8.4 SPLIT PARTITION

Divide one partition into two. Useful when a partition grew unexpectedly large.

```sql
-- ============================================================
-- Scenario: p_future grew large because interval partitioning
-- was not set up. Split it into two defined partitions.
--
-- Before: p_future covers DATE '2025-06-01' to MAXVALUE
-- After:  p_2025_06 covers to '2025-07-01'
--         p_future  covers '2025-07-01' to MAXVALUE
-- ============================================================
ALTER TABLE ORDERS
SPLIT PARTITION p_future
    AT (DATE '2025-07-01')
    INTO (
        PARTITION p_2025_06 TABLESPACE tbs_current,
        PARTITION p_future  TABLESPACE tbs_current
    )
UPDATE INDEXES;
```

---

### 8.5 MERGE PARTITIONS

Combine two adjacent partitions into one. Useful for consolidating low-volume historical partitions.

```sql
-- ============================================================
-- Merge Jan and Feb 2023 into a single Q1_2023_JAN_FEB partition.
-- Rows from both partitions move into the new merged partition.
-- ============================================================
ALTER TABLE ORDERS
MERGE PARTITIONS p_2023_01, p_2023_02
INTO PARTITION p_2023_q1_jan_feb
    TABLESPACE tbs_hist_2023
UPDATE INDEXES;
```

---

### 8.6 EXCHANGE PARTITION

Swap a partition with a regular (non-partitioned) table of identical structure. This is the fastest way to load data into a partition: load to a staging table without the overhead of the partitioned table, then exchange.

```sql
-- ============================================================
-- STEP 1: Create a staging table with identical structure
-- (no partitioning, no FK constraints for fast load).
-- ============================================================
CREATE TABLE ORDERS_STAGING_2025_05 AS
SELECT * FROM ORDERS WHERE 1 = 0;  -- Empty copy of structure

-- STEP 2: Load data into staging table (fast, bulk insert)
INSERT /*+ APPEND */ INTO ORDERS_STAGING_2025_05
(ORDER_ID, ORDER_DATE, CUSTOMER_ID, REGION, STATUS,
 TOTAL_AMOUNT, CURRENCY_CODE, PAYMENT_STATUS)
SELECT
    800 + LEVEL,
    DATE '2025-05-01' + MOD(LEVEL, 30),
    MOD(LEVEL, 8) + 1,
    CASE MOD(LEVEL, 5)
        WHEN 0 THEN 'NORTH' WHEN 1 THEN 'SOUTH'
        WHEN 2 THEN 'EAST'  WHEN 3 THEN 'WEST'
        ELSE 'INTERNATIONAL'
    END,
    'CONFIRMED',
    ROUND(DBMS_RANDOM.VALUE(100, 5000), 2),
    'USD',
    'UNPAID'
FROM DUAL CONNECT BY LEVEL <= 300;

COMMIT;

-- STEP 3: Exchange the staging table with the target partition.
-- INCLUDING INDEXES: exchange index data too.
-- WITHOUT VALIDATION: trust the data; skip row-by-row validation
--   (safe when you control the staging load).
-- UPDATE INDEXES: keep global indexes usable.
ALTER TABLE ORDERS
EXCHANGE PARTITION p_2025_05
WITH TABLE ORDERS_STAGING_2025_05
INCLUDING INDEXES
WITHOUT VALIDATION
UPDATE INDEXES;

-- After exchange:
-- - p_2025_05 partition now contains the staging data (instantly)
-- - ORDERS_STAGING_2025_05 is now empty (or contains old partition data)
-- STEP 4: Drop the staging table
DROP TABLE ORDERS_STAGING_2025_05;
```

---

### 8.7 REBUILD INDEX PARTITION

After a data load or partition operation, an index partition may need rebuilding.

```sql
-- ============================================================
-- Rebuild a specific local index partition after bulk load.
-- Only affects the index segment for p_2025_05 — other index
-- partitions are untouched and remain available.
-- ============================================================
ALTER INDEX idx_orders_customer_local
REBUILD PARTITION p_2025_05
ONLINE            -- Allows concurrent DML during rebuild
PARALLEL 4;       -- Use 4 parallel execution servers

-- Check status after rebuild
SELECT PARTITION_NAME, STATUS
FROM USER_IND_PARTITIONS
WHERE INDEX_NAME = 'IDX_ORDERS_CUSTOMER_LOCAL'
  AND PARTITION_NAME = 'P_2025_05';
```

---

### 8.8 Updating Optimizer Statistics

Accurate statistics are critical for the optimizer to make good partition pruning decisions and choose efficient execution plans.

```sql
-- ============================================================
-- Gather statistics for the entire ORDERS table.
-- Cascade: also gathers stats on all indexes.
-- Degree 4: use 4 parallel workers.
-- ============================================================
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS(
        ownname          => USER,
        tabname          => 'ORDERS',
        cascade          => TRUE,
        degree           => 4,
        method_opt       => 'FOR ALL COLUMNS SIZE AUTO',
        granularity      => 'ALL'   -- Stats for table, partitions, subpartitions
    );
END;
/

-- ============================================================
-- Gather stats for a single partition only (faster for
-- targeted maintenance after bulk load into one partition).
-- ============================================================
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS(
        ownname          => USER,
        tabname          => 'ORDERS',
        partname         => 'P_2025_05',
        cascade          => TRUE,
        degree           => 4,
        granularity      => 'PARTITION'
    );
END;
/

-- ============================================================
-- Check when statistics were last gathered
-- ============================================================
SELECT
    TABLE_NAME,
    PARTITION_NAME,
    NUM_ROWS,
    LAST_ANALYZED
FROM USER_TAB_PARTITIONS
WHERE TABLE_NAME = 'ORDERS'
ORDER BY PARTITION_POSITION DESC
FETCH FIRST 5 ROWS ONLY;
```

---

## 9. Performance Tuning with EXPLAIN PLAN

### 9.1 Generating an Execution Plan

```sql
-- ============================================================
-- Step 1: Populate the PLAN_TABLE with the execution plan.
-- ============================================================
EXPLAIN PLAN FOR
SELECT
    ORDER_ID,
    ORDER_DATE,
    CUSTOMER_ID,
    TOTAL_AMOUNT
FROM ORDERS
WHERE ORDER_DATE >= DATE '2024-01-01'
  AND ORDER_DATE <  DATE '2024-02-01'
  AND REGION = 'NORTH';

-- Step 2: Display the plan in readable format.
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(FORMAT => 'ALL'));
```

**Expected plan output (with partition pruning active):**

```
Plan hash value: 3847261902

----------------------------------------------------------------------------------------------------------------
| Id  | Operation                              | Name   | Rows | Bytes | Cost | Pstart | Pstop |
----------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                       |        |   89 | 4450  |   12 |        |       |
|   1 |  PARTITION RANGE SINGLE                |        |   89 | 4450  |   12 |     14 |    14 |
|   2 |   PARTITION LIST SINGLE                |        |   89 | 4450  |   12 |      1 |     1 |
|   3 |    TABLE ACCESS FULL                   | ORDERS |   89 | 4450  |   12 |     14 |    14 |
----------------------------------------------------------------------------------------------------------------

Pstart = 14  → Partition position 14 (p_2024_01)
Pstop  = 14  → Same partition — only one partition accessed
PARTITION RANGE SINGLE → Range pruning to single partition
PARTITION LIST SINGLE  → List pruning to single sub-partition
```

> **Reading the plan**: `Pstart` and `Pstop` tell you which partition positions Oracle accesses. When they are equal, only one partition is scanned. When they show `KEY` or `KEY(I)`, pruning is dynamic (bind variable driven). When they show `1` to `N` (total partitions), no pruning occurred — investigate your WHERE clause.

---

### 9.2 Verifying Pruning Fails — Applying a Function on Partition Key

```sql
-- ============================================================
-- BAD QUERY — Partition pruning is DISABLED.
-- TO_CHAR wraps ORDER_DATE, so Oracle cannot evaluate the
-- partition bound condition at parse or execute time.
-- Result: FULL TABLE SCAN across ALL partitions.
-- ============================================================
EXPLAIN PLAN FOR
SELECT ORDER_ID, ORDER_DATE, TOTAL_AMOUNT
FROM ORDERS
WHERE TO_CHAR(ORDER_DATE, 'YYYY-MM') = '2024-01';  -- ← WRONG

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(FORMAT => 'ALL'));
```

**Expected plan (no pruning):**

```
-------------------------------------------------------------------------------------------
| Id  | Operation          | Name   | Rows  | Bytes | Cost  | Pstart | Pstop |
-------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |        |  2450 | 122K  | 8234  |        |       |
|   1 |  PARTITION RANGE ALL|       |  2450 | 122K  | 8234  |      1 |    30 |  ← ALL 30 partitions!
|   2 |   TABLE ACCESS FULL| ORDERS |  2450 | 122K  | 8234  |      1 |    30 |
-------------------------------------------------------------------------------------------
```

`Pstart = 1, Pstop = 30` — all 30 partitions are scanned. This is the opposite of pruning. The fix:

```sql
-- GOOD QUERY — Pruning restored.
SELECT ORDER_ID, ORDER_DATE, TOTAL_AMOUNT
FROM ORDERS
WHERE ORDER_DATE >= DATE '2024-01-01'
  AND ORDER_DATE <  DATE '2024-02-01';  -- ← Correct: raw date comparison
```

---

### 9.3 Dynamic Pruning with Bind Variables

```sql
-- ============================================================
-- With bind variables, Oracle cannot determine the exact
-- partition at parse time, so pruning happens at EXECUTE time.
-- The plan shows Pstart = KEY, Pstop = KEY.
-- This is still efficient — Oracle prunes at runtime.
-- ============================================================
VARIABLE v_start VARCHAR2(20);
VARIABLE v_end   VARCHAR2(20);
EXEC :v_start := '2024-01-01';
EXEC :v_end   := '2024-02-01';

EXPLAIN PLAN FOR
SELECT ORDER_ID, ORDER_DATE, TOTAL_AMOUNT
FROM ORDERS
WHERE ORDER_DATE >= TO_DATE(:v_start, 'YYYY-MM-DD')
  AND ORDER_DATE <  TO_DATE(:v_end,   'YYYY-MM-DD');

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
-- Pstart = KEY, Pstop = KEY → Dynamic pruning — still good.
```

---

### 9.4 Common Reasons Partition Pruning Fails

| Cause | Bad Example | Fix |
|---|---|---|
| Function applied to partition key | `TO_CHAR(ORDER_DATE, 'YYYY-MM') = '2024-01'` | Use raw date range: `ORDER_DATE >= DATE '2024-01-01' AND ORDER_DATE < DATE '2024-02-01'` |
| Implicit type conversion | `WHERE ORDER_DATE = '01-JAN-24'` (string) | Use `DATE '2024-01-01'` literal or `TO_DATE(...)` explicitly |
| TRUNC on partition key | `WHERE TRUNC(ORDER_DATE) = DATE '2024-01-15'` | Use `ORDER_DATE >= DATE '2024-01-15' AND ORDER_DATE < DATE '2024-01-16'` |
| Not filtering by partition key | `WHERE CUSTOMER_ID = 5` (no date) | Add date range condition |
| Non-sargable conditions | `WHERE ORDER_DATE - 30 < SYSDATE` | Rewrite as `ORDER_DATE < SYSDATE + 30` |
| OR across partition key | `WHERE ORDER_DATE = DATE '2024-01-01' OR STATUS = 'PENDING'` | Use UNION ALL for separate partition filters |
| Subquery on partition key | `WHERE ORDER_DATE IN (SELECT MAX(D) FROM ...)` | Pre-compute the value and use a bind variable or literal |

---

## 10. Laravel Developer Integration Guide

### Why Laravel Developers Need to Understand Partitioning

When a Laravel application is backed by an Oracle database with partitioned tables, the application code itself does not change — partitioning is completely transparent at the SQL level. However, the **way you write your queries** determines whether Oracle can prune partitions or must scan everything.

A poorly written Eloquent query on a 500-million-row partitioned table can be 60× slower than a properly written one — not because of Laravel, but because the WHERE clause prevents partition pruning.

---

### How Partitioning Helps Laravel Applications

| Laravel Use Case | Table | Partitioning Benefit |
|---|---|---|
| Order history pages | orders | Monthly pruning: "last 3 months" scans 3/60 partitions |
| Payment log export | payments | Prune to billing period only |
| Audit trail search | audit_logs | List prune by event type instantly |
| Activity log dashboards | activity_logs | Date-range prune: last 7 or 30 days |
| Notification history | notifications | Monthly pruning for old notification cleanup |
| Admin report generation | order_items + orders | Composite prune: month + region |

---

### Example 1 — Correct DB::select() with Partition-Pruning Query

```php
<?php
// ============================================================
// Monthly sales report — partition pruning via raw date range.
// This generates a query that Oracle can prune to 12 partitions
// (one year) without function application on ORDER_DATE.
// ============================================================

use Illuminate\Support\Facades\DB;

$year = 2024;

$monthlySales = DB::select("
    SELECT
        TO_CHAR(ORDER_DATE, 'YYYY-MM')  AS sales_month,
        REGION,
        COUNT(*)                         AS total_orders,
        SUM(TOTAL_AMOUNT)                AS gross_revenue,
        ROUND(AVG(TOTAL_AMOUNT), 2)      AS avg_order_value
    FROM ORDERS
    WHERE ORDER_DATE >= TO_DATE(:start_date, 'YYYY-MM-DD')
      AND ORDER_DATE <  TO_DATE(:end_date,   'YYYY-MM-DD')
      AND STATUS != 'CANCELLED'
    GROUP BY TO_CHAR(ORDER_DATE, 'YYYY-MM'), REGION
    ORDER BY sales_month, REGION
", [
    'start_date' => $year . '-01-01',
    'end_date'   => ($year + 1) . '-01-01',
]);
```

> Note: The `TO_CHAR` is in the `SELECT` list (for display only), not in the `WHERE` clause. Partition pruning is applied to the raw `ORDER_DATE` comparison.

---

### Example 2 — Correct Eloquent Query Builder with Date Range

```php
<?php
// ============================================================
// Get a customer's orders from the last 3 months.
// Uses Carbon to build literal date bounds — Oracle receives
// a raw date comparison without functions on ORDER_DATE.
// ============================================================

use App\Models\Order;
use Carbon\Carbon;

$customerId = 42;
$startDate  = Carbon::now()->subMonths(3)->startOfMonth()->format('Y-m-d');
$endDate    = Carbon::now()->format('Y-m-d');

$orders = Order::where('customer_id', $customerId)
    ->whereBetween('order_date', [$startDate, $endDate])
    ->where('status', '!=', 'CANCELLED')
    ->orderBy('order_date', 'desc')
    ->paginate(25);

// The generated SQL becomes:
// SELECT * FROM ORDERS
// WHERE CUSTOMER_ID = 42
//   AND ORDER_DATE BETWEEN TO_DATE('2024-03-01','YYYY-MM-DD')
//                      AND TO_DATE('2024-05-31','YYYY-MM-DD')
//   AND STATUS != 'CANCELLED'
// ORDER BY ORDER_DATE DESC
// This triggers partition pruning to only the 3 relevant partitions.
```

---

### Example 3 — Partition-Aware Audit Log Query

```php
<?php
// ============================================================
// Fetch ORDER audit events from the last 30 days.
// List partition pruning on EVENT_TYPE = 'ORDER'.
// Date index on EVENT_DATE further narrows the scan.
// ============================================================

use Illuminate\Support\Facades\DB;

$auditLogs = DB::select("
    SELECT
        LOG_ID,
        EVENT_DATE,
        ACTION,
        ENTITY_ID,
        USER_ID,
        STATUS
    FROM AUDIT_LOGS
    WHERE EVENT_TYPE = :event_type
      AND EVENT_DATE >= :since_date
    ORDER BY EVENT_DATE DESC
    FETCH FIRST :limit ROWS ONLY
", [
    'event_type' => 'ORDER',
    'since_date' => Carbon\Carbon::now()->subDays(30)->format('Y-m-d'),
    'limit'      => 100,
]);
```

---

### Common Laravel Mistakes with Partitioned Oracle Tables

#### Mistake 1 — Using TO_CHAR in WHERE clause

```php
// ❌ WRONG — Applies TO_CHAR to ORDER_DATE, kills partition pruning.
// Oracle cannot evaluate partition bounds against a string expression.
$orders = DB::select("
    SELECT * FROM ORDERS
    WHERE TO_CHAR(ORDER_DATE, 'YYYY-MM') = :month
", ['month' => '2024-01']);

// ✅ CORRECT — Use raw date range bounds.
$orders = DB::select("
    SELECT * FROM ORDERS
    WHERE ORDER_DATE >= TO_DATE(:start, 'YYYY-MM-DD')
      AND ORDER_DATE <  TO_DATE(:end,   'YYYY-MM-DD')
", [
    'start' => '2024-01-01',
    'end'   => '2024-02-01',
]);
```

---

#### Mistake 2 — Loading All Historical Data Without Pagination

```php
// ❌ WRONG — Loads all orders ever — 500M rows — into memory.
$allOrders = Order::where('customer_id', $id)->get();

// ✅ CORRECT — Add date range + pagination.
$orders = Order::where('customer_id', $id)
    ->whereBetween('order_date', [
        now()->subMonths(6)->startOfMonth()->toDateString(),
        now()->toDateString(),
    ])
    ->orderBy('order_date', 'desc')
    ->paginate(50);
```

---

#### Mistake 3 — TRUNC on Partition Key

```php
// ❌ WRONG — TRUNC(ORDER_DATE) prevents pruning.
$orders = DB::select("
    SELECT * FROM ORDERS WHERE TRUNC(ORDER_DATE) = :date
", ['date' => '2024-01-15']);

// ✅ CORRECT — Range condition.
$orders = DB::select("
    SELECT * FROM ORDERS
    WHERE ORDER_DATE >= TO_DATE(:start, 'YYYY-MM-DD')
      AND ORDER_DATE <  TO_DATE(:end,   'YYYY-MM-DD')
", [
    'start' => '2024-01-15',
    'end'   => '2024-01-16',
]);
```

---

#### Mistake 4 — Assuming Partitioning Replaces Indexing

Partitioning prunes at the **partition level** (e.g., one month = millions of rows). Within that partition, Oracle still needs an index to find individual rows efficiently. Always add appropriate local indexes in addition to partitioning.

```php
// Even with partitioning, this query benefits from idx_orders_customer_local:
$orders = Order::where('customer_id', 999)
    ->whereBetween('order_date', ['2024-01-01', '2024-02-01'])
    ->get();
// Oracle prunes to one partition (date range), then uses the customer
// local index within that partition to find only customer 999's rows.
```

---

#### Mistake 5 — Ignoring Execution Plans

Laravel developers should routinely check Oracle execution plans during development:

```php
// In development/testing — inspect the plan for a slow query.
$plan = DB::select("
    SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST'))
");
dd($plan);
```

Or enable Oracle SQL tracing and analyze with TKPROF for production investigations.

---

## 11. Practical Real-Life Use Cases

| Industry | Table | Partition Strategy | Reason |
|---|---|---|---|
| E-Commerce | `orders`, `order_items` | Range (date) + List (region) | Monthly reporting, regional fulfillment |
| Banking | `transaction_history` | Range (txn_date) monthly | Regulatory retention, period reporting |
| Telecom | `call_detail_records` | Range (call_date) daily/monthly | Billions of records per day, CDR billing |
| Healthcare | `patient_visits` | Range (visit_date) yearly | HIPAA retention, historical reports |
| Government | `tax_filings` | Range (filing_year) + List (state) | Annual tax periods, state-by-state reports |
| Payment Gateway | `payment_logs` | Range (log_date) + List (method) | PCI-DSS compliance, method performance |
| ERP | `inventory_movements` | Range (movement_date) quarterly | Periodic audits, quarter-close reports |
| Education | `student_results` | Range (academic_year) | Year-wise archive and report isolation |
| Logistics | `shipment_tracking` | Range (ship_date) + Hash (carrier) | Even load across carriers, time-based reports |
| Social/Media | `activity_logs` | Range (activity_date) + List (event_type) | Event type isolation, GDPR deletion |

---

## 12. Comparison Tables

### Normal Table vs Partitioned Table

| Feature | Normal Table | Partitioned Table |
|---|---|---|
| Physical structure | Single segment | One segment per partition |
| Full table scan | Reads entire table | Reads only pruned partitions |
| DELETE for archival | Row-by-row, slow, massive redo | DROP/TRUNCATE PARTITION, instant |
| Index rebuilds | Rebuild entire index | Rebuild only one index partition |
| Parallel query | Limited | Natural — one thread per partition |
| Tablespace per period | Not possible | Yes — old data on cheap storage |
| Availability | All-or-nothing | Per-partition operations possible |
| Complexity | Low | Medium–High (requires design) |
| When to use | < 50M rows, no time-range queries | > 50M rows, time-range queries, archival needs |

---

### Range vs List vs Hash vs Composite vs Interval vs Reference

| Strategy | Key Characteristic | Best For | Auto-Creates Partitions? |
|---|---|---|---|
| Range | Contiguous value ranges | Dates, numeric IDs | No (Yes with Interval) |
| List | Discrete enumerated values | Region, country, category | No |
| Hash | Hash function on key | Even distribution, no natural range | No |
| Composite | Two-level (Range+List, Range+Hash) | Date + region, date + category | Partially |
| Interval | Range with auto-extension | Growing time-series | **Yes** |
| Reference | Inherited from parent via FK | Child tables (order_items) | Follows parent |

---

### Local Index vs Global Index

| Feature | Local Index | Global Index |
|---|---|---|
| Structure | One index partition per table partition | Single index spanning all partitions |
| Co-location | Index and data in same partition | Index separate from data |
| DROP/TRUNCATE partition | Index partition drops automatically | Global index becomes UNUSABLE |
| Rebuild scope | One partition only | Full index (expensive) |
| Uniqueness | Can enforce uniqueness only if PK contains partition key | Can enforce uniqueness across all partitions |
| Query performance | Excellent when query prunes partitions | Better when query cannot prune |
| Maintenance overhead | Low | Higher |
| Best for | Partition maintenance–heavy workloads | Lookups by non-partition-key unique column |

---

### Partitioning vs Indexing

| | Partitioning | Indexing |
|---|---|---|
| Granularity | Partition level | Row level |
| Best query type | Range scans, bulk operations | Point lookups, selective predicates |
| Maintenance cost | Very low per partition | High as table grows |
| Space overhead | Minimal | Significant (B-tree nodes) |
| DML impact | Low | INSERT/UPDATE/DELETE maintain index |
| Replaces the other? | No | No |
| Combined use | Yes — always use both | Yes — always use both |

---

### Partition Maintenance Commands

| Command | What It Does | When to Use |
|---|---|---|
| `ADD PARTITION` | Add a new partition | Before a new time period begins |
| `DROP PARTITION` | Delete partition + data instantly | Data past retention policy |
| `TRUNCATE PARTITION` | Remove data, keep partition shell | Reload/reprocess a period |
| `SPLIT PARTITION` | Divide one partition into two | Partition grew too large |
| `MERGE PARTITIONS` | Combine two partitions into one | Consolidate low-volume old data |
| `EXCHANGE PARTITION` | Swap partition ↔ staging table | Fastest bulk load technique |
| `REBUILD PARTITION` | Rebuild one index partition | After bulk load or UNUSABLE status |
| `GATHER_TABLE_STATS` | Update optimizer statistics | After any DML or partition operation |

---

## 13. Common Mistakes and How to Avoid Them

### 1. Choosing the Wrong Partition Key

**Mistake:** Partitioning on a low-cardinality column (e.g., STATUS with 5 values) using range partitioning, or choosing a column that is never used in WHERE clauses.

**Fix:** Choose the column most commonly used in range filters. For time-series data, always prefer the date/timestamp column.

---

### 2. Creating Too Many Small Partitions

**Mistake:** Daily partitions on a table with only 10,000 rows/day. 365 partitions × 5 years = 1,825 partitions, each tiny. Oracle's partition metadata management overhead grows.

**Fix:** Match partition granularity to query patterns and data volume. Monthly partitions for 1–50M rows/month. Daily only if rows/day are in the millions.

---

### 3. Creating Very Large Partitions

**Mistake:** One partition per year on a table with 500M rows/year. Pruning eliminates other years but still scans 500M rows in one partition.

**Fix:** Partition granularity should mean each partition is workable — typically 100M rows maximum per partition for comfortable maintenance.

---

### 4. Not Using Date Range Filters

**Mistake:** Querying the partitioned table without the partition key in the WHERE clause, causing a full scan of all partitions.

**Fix:** Every query against a partitioned table should include a filter on the partition key, or accept that partition pruning will not occur.

---

### 5. Applying Functions on Partition Columns

**Mistake:** `WHERE TRUNC(ORDER_DATE) = :date`, `WHERE TO_CHAR(ORDER_DATE, ...) = :str`, `WHERE EXTRACT(YEAR FROM ORDER_DATE) = 2024`.

**Fix:** Always use raw date range comparisons: `ORDER_DATE >= x AND ORDER_DATE < y`.

---

### 6. Forgetting Global Index Implications

**Mistake:** Dropping a partition without `UPDATE INDEXES`. All global indexes become `UNUSABLE` — application queries fail with `ORA-01502`.

**Fix:** Always include `UPDATE INDEXES` in `DROP PARTITION` and `TRUNCATE PARTITION` commands, or rebuild global indexes immediately after.

---

### 7. Not Updating Statistics After Partition Operations

**Mistake:** Loading 5M rows into a partition via EXCHANGE, then running reports. The optimizer still thinks the partition has 0 rows (stale stats) and chooses a wrong plan.

**Fix:** Always run `DBMS_STATS.GATHER_TABLE_STATS` after significant data changes.

---

### 8. Thinking Partitioning Automatically Improves Every Query

**Mistake:** Assuming any query on a partitioned table is fast. A query without the partition key filter scans all partitions — potentially slower than a well-indexed non-partitioned table.

**Fix:** Partitioning is not a magic fix. It helps specific query patterns. Design your partition key around your most critical query filters.

---

### 9. Mixing Partitioning with a Poor Data Model

**Mistake:** Partitioning a table that has no logical time-based query pattern, or partitioning a small table that should just be indexed.

**Fix:** Only partition tables where the volume and query patterns genuinely benefit. A 1M-row table with a good index needs no partitioning.

---

### 10. Using Partitioning When a Normal Indexed Table Is Enough

**Mistake:** Adding partitioning complexity to a 5M-row table that fits comfortably in the buffer cache and performs fine with indexes.

**Fix:** Oracle recommendation: consider partitioning when a table exceeds 50–100M rows, or when maintenance operations (archiving, bulk loads) become painful on a non-partitioned table.

---

## 14. Best Practices

- **Design partitioning around actual query patterns, not just table size.** Profile your most frequent and critical queries. If they all filter by `ORDER_DATE`, partition by date. If they filter by region, consider list or composite.

- **Prefer interval partitioning for growing time-series tables.** It eliminates the maintenance burden of manually adding new partitions each month. Combine with a scheduled job to pre-create near-future partitions as a safety buffer.

- **Use composite partitioning (Range-List) when two dimensions are common query filters.** Monthly + regional queries prune to a single sub-partition — the most aggressive pruning possible.

- **Keep partition sizes manageable — target 50M–200M rows per partition.** This ensures partition-level operations (TRUNCATE, SPLIT, EXCHANGE) complete in reasonable time and keeps execution plans predictable.

- **Use local indexes whenever partition maintenance operations are frequent.** Local indexes are self-maintaining — they follow the partition structure automatically. Global indexes require careful handling on every partition operation.

- **Use `UPDATE INDEXES` on every `DROP PARTITION` and `TRUNCATE PARTITION`.** This prevents global indexes from becoming UNUSABLE and avoids application-breaking errors.

- **Gather optimizer statistics after every significant data change.** Use `DBMS_STATS.GATHER_TABLE_STATS` with `granularity => 'ALL'` to update table, partition, and sub-partition statistics.

- **Validate query plans with `EXPLAIN PLAN` and `DBMS_XPLAN.DISPLAY`.** Always verify that new queries trigger partition pruning before deploying to production. Check `Pstart` and `Pstop` values.

- **Use EXCHANGE PARTITION for large bulk loads.** Load data into a staging table first, then exchange it into the partitioned table in seconds. This minimizes impact on the live system.

- **Assign historical partitions to cheaper tablespaces.** Use Oracle's ILM (Information Lifecycle Management) features to automatically move old partitions to compressed, read-only, or slower storage tiers.

- **Monitor partition growth with `USER_TAB_PARTITIONS`.** Track `NUM_ROWS`, `BLOCKS`, and `LAST_ANALYZED` regularly. Set up alerts if a partition grows unexpectedly large (indicates interval or split issues).

- **Test pruning with real bind variables in realistic conditions.** Dynamic pruning (`Pstart = KEY`) is safe, but test that runtime variable binding does not cause full scans due to type mismatches.

---

## 15. Conclusion

Oracle table partitioning is one of the most powerful tools available for managing large-scale enterprise databases. When designed correctly around real query patterns, it transforms slow, painful table scans and maintenance windows into fast, surgical partition-level operations.

The key principles to remember:

- **Partition pruning is the primary performance benefit** — and it only works when your WHERE clause uses the raw partition key with direct comparison operators.
- **Partition maintenance (DROP, TRUNCATE, EXCHANGE) is the primary operational benefit** — archiving or retiring a year of data becomes an instant metadata operation instead of a multi-hour DELETE job.
- **Local indexes are your first choice** — they are self-maintaining and perfectly aligned with the partition structure.
- **Partitioning and indexing are complementary** — partitioning prunes to the right partition; indexes find the right rows within it.
- **For Laravel developers**: the application code stays the same — but the WHERE clause you write determines whether Oracle reads 1 partition or 60. Avoid applying functions to partition key columns in any WHERE condition.

Used thoughtfully, partitioning enables Oracle databases to handle billions of rows with query response times measured in seconds, maintenance windows measured in minutes, and storage costs managed through intelligent data tiering — all essential properties for production-grade enterprise systems.

---

*Document Version: 1.0 | Oracle Database 19c / 21c syntax | Compatible with Oracle Enterprise Edition*

*Tested with: Oracle Database 19c Enterprise Edition, Oracle Linux 8*
