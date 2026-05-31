# Oracle `CROSS APPLY` and `OUTER APPLY`: A Complete Advanced Guide

> A production-grade reference for Oracle developers and Laravel engineers working with correlated, row-limited, and per-group SQL queries.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [What Is APPLY in SQL?](#2-what-is-apply-in-sql)
3. [Why Is APPLY Useful in Oracle?](#3-why-is-apply-useful-in-oracle)
4. [How the Right-Side Query References Left-Side Columns](#4-how-the-right-side-query-references-left-side-columns)
5. [Oracle Version Support](#5-oracle-version-support)
6. [Core Concepts](#6-core-concepts)
7. [Basic Syntax Explained](#7-basic-syntax-explained)
8. [Database Setup: E-Commerce Schema](#8-database-setup-e-commerce-schema)
9. [CROSS APPLY Examples](#9-cross-apply-examples)
   - [9.1 Latest Order Per Customer](#91-latest-order-per-customer)
   - [9.2 Top 3 Products Per Order](#92-top-3-products-per-order)
   - [9.3 Latest Successful Payment Per Order](#93-latest-successful-payment-per-order)
   - [9.4 Customer's Most Recent Support Ticket](#94-customers-most-recent-support-ticket)
   - [9.5 Correlated Aggregation Per Customer](#95-correlated-aggregation-per-customer)
10. [OUTER APPLY Examples](#10-outer-apply-examples)
    - [10.1 All Customers With Latest Order If Available](#101-all-customers-with-latest-order-if-available)
    - [10.2 All Orders With Latest Payment If Available](#102-all-orders-with-latest-payment-if-available)
    - [10.3 All Customers With Last Support Ticket If Available](#103-all-customers-with-last-support-ticket-if-available)
    - [10.4 Customer Dashboard Query](#104-customer-dashboard-query)
    - [10.5 Product Sales Summary](#105-product-sales-summary)
11. [APPLY vs Other SQL Techniques](#11-apply-vs-other-sql-techniques)
    - [11.1 CROSS APPLY vs INNER JOIN](#111-cross-apply-vs-inner-join)
    - [11.2 OUTER APPLY vs LEFT JOIN](#112-outer-apply-vs-left-join)
    - [11.3 APPLY vs Correlated Subquery](#113-apply-vs-correlated-subquery)
    - [11.4 APPLY vs Window Functions](#114-apply-vs-window-functions)
12. [LATERAL Inline Views in Oracle](#12-lateral-inline-views-in-oracle)
13. [Advanced Business Scenario: Admin Dashboard](#13-advanced-business-scenario-admin-dashboard)
14. [Performance Tuning](#14-performance-tuning)
15. [Laravel Developer Guide](#15-laravel-developer-guide)
16. [Practical Real-World Use Cases](#16-practical-real-world-use-cases)
17. [Comparison Tables](#17-comparison-tables)
18. [Common Mistakes](#18-common-mistakes)
19. [Best Practices](#19-best-practices)
20. [Conclusion](#20-conclusion)

---

## 1. Introduction

`CROSS APPLY` and `OUTER APPLY` are powerful Oracle SQL constructs that let you execute a correlated subquery **per row** of a left-side table and treat the result as an inline table. They solve a specific class of problems that standard joins and correlated subqueries handle poorly: **row-limited, ordered, or aggregated results per group**.

Classic use cases include:

- The **latest order** for each customer
- The **top 3 products** by revenue for each order
- The **most recent payment status** for each invoice
- A **multi-column per-row calculation** without repeating the subquery four times

This guide explains both constructs at an advanced level, builds a complete e-commerce database to demonstrate every example, compares them against joins, window functions, and `LATERAL`, and provides a dedicated section for Laravel developers querying Oracle databases.

---

## 2. What Is APPLY in SQL?

`APPLY` is a **lateral join operator**. The word *lateral* means the right-side expression is allowed to reference columns from the left-side table — something a regular join cannot do directly.

Conceptually, `APPLY` works like this:

```
For each row in the LEFT table:
    Execute the RIGHT-side subquery using that row's column values
    Attach the result rows to the current left row
```

This is called a **correlated inline view** — an inline view (subquery in the FROM clause) whose WHERE clause references the outer table.

SQL Server introduced `CROSS APPLY` / `OUTER APPLY` in 2005. Oracle added the `LATERAL` keyword in Oracle 12c and later added `CROSS APPLY` / `OUTER APPLY` as aliases for the same feature. PostgreSQL uses `LATERAL` exclusively. They are all expressions of the same concept: a **per-row correlated table expression**.

---

## 3. Why Is APPLY Useful in Oracle?

Standard SQL joins cannot contain `ORDER BY` or `FETCH FIRST` inside the join condition. A correlated subquery in the `SELECT` list can only return a **single column and a single value**. Neither approach cleanly handles these requirements:

- **Top-N rows per group** — e.g., latest 3 transactions per account
- **Row-limited ordered results** — e.g., the single most recent payment per order
- **Multiple calculated columns from one subquery** — e.g., total orders, total spend, and last order date all in one pass
- **Complex per-row logic** — e.g., conditional aggregation scoped to each parent row

`APPLY` solves all four cleanly. The right-side subquery can use `ORDER BY`, `FETCH FIRST`, aggregation, `CASE` logic, and multiple columns — all evaluated freshly for each left-side row.

---

## 4. How the Right-Side Query References Left-Side Columns

The defining feature of `APPLY` is that **the right-side subquery sees the current row's columns from the left-side table**. This is the lateral reference.

```sql
SELECT c.customer_id, c.full_name, latest.order_date
FROM customers c                          -- (A) left-side table
CROSS APPLY (                             -- (B) right-side subquery
    SELECT o.order_date
    FROM orders o
    WHERE o.customer_id = c.customer_id   -- (C) lateral reference to (A)
    ORDER BY o.order_date DESC
    FETCH FIRST 1 ROW ONLY
) latest;                                 -- (D) alias for the right-side result
```

- **(A)** `customers c` — the driving table; Oracle iterates through each row here
- **(B)** The subquery block after `CROSS APPLY` or `OUTER APPLY`
- **(C)** `c.customer_id` inside the subquery refers to the **current row** of `customers` — this is the lateral reference that makes it correlated
- **(D)** `latest` is the alias for the subquery's result columns, used in the outer `SELECT`

Each time Oracle processes a row from `customers`, it re-executes the inner subquery with that row's `customer_id`, fetching only the most recent order for that specific customer. Without `APPLY`, achieving this with `ORDER BY` and `FETCH FIRST` inside a join is not possible in standard Oracle SQL.

---

## 5. Oracle Version Support

| Feature | Minimum Oracle Version | Notes |
|---|---|---|
| `LATERAL` keyword | Oracle 12c R1 (12.1.0.1) | Requires `FROM t1, LATERAL (...)` syntax |
| `CROSS APPLY` | Oracle 12c R2 (12.2.0.1) | Alias for `CROSS JOIN LATERAL` |
| `OUTER APPLY` | Oracle 12c R2 (12.2.0.1) | Alias for `LEFT JOIN LATERAL ... ON 1=1` |
| `FETCH FIRST N ROWS ONLY` | Oracle 12c R1 (12.1.0.1) | Required inside `APPLY` subqueries |
| `GENERATED AS IDENTITY` | Oracle 12c R1 (12.1.0.1) | Used in the schema below |

**Recommendation:** Use Oracle 19c or later in production. All examples in this guide run on Oracle 12c R2 and above. If you are on Oracle 11g or earlier, use the `LATERAL` workaround available in 11g via the undocumented `/*+ NO_MERGE */` hint or rewrite using `ROW_NUMBER()`.

---

## 6. Core Concepts

### `CROSS APPLY` — Inner Join Behaviour

`CROSS APPLY` returns **only left-side rows where the right-side subquery produces at least one row**. If the subquery returns no rows for a given left-side row, that left-side row is **excluded** from the result. This mirrors an `INNER JOIN`.

```
customer has orders     → included  (subquery returns rows)
customer has NO orders  → excluded  (subquery returns zero rows)
```

### `OUTER APPLY` — Left Join Behaviour

`OUTER APPLY` returns **all left-side rows**, whether or not the right-side subquery produces rows. If the subquery returns no rows for a given left-side row, the right-side columns appear as `NULL`. This mirrors a `LEFT OUTER JOIN`.

```
customer has orders     → included  (right-side columns populated)
customer has NO orders  → included  (right-side columns are NULL)
```

### The Key Difference at a Glance

```
CROSS APPLY  ≈  INNER JOIN  +  correlated inline view  +  ORDER BY / FETCH FIRST
OUTER APPLY  ≈  LEFT JOIN   +  correlated inline view  +  ORDER BY / FETCH FIRST
```

The `ORDER BY` and `FETCH FIRST` inside the subquery are what make `APPLY` irreplaceable for top-N per group queries.

---

## 7. Basic Syntax Explained

### `CROSS APPLY` Syntax

```sql
SELECT
    p.parent_id,
    p.parent_name,
    child_result.child_col1,
    child_result.child_col2
FROM parent_table p
CROSS APPLY (
    SELECT c.child_col1, c.child_col2
    FROM child_table c
    WHERE c.parent_id = p.parent_id   -- lateral reference
    ORDER BY c.some_date DESC          -- allowed inside APPLY
    FETCH FIRST 1 ROW ONLY             -- top-N per parent row
) child_result                         -- alias for subquery result
ORDER BY p.parent_id;
```

**Step-by-step breakdown:**

1. `FROM parent_table p` — Oracle starts by scanning the left table
2. `CROSS APPLY (...)` — for each row from `parent_table`, execute the subquery
3. `WHERE c.parent_id = p.parent_id` — the lateral reference ties the subquery to the current parent row
4. `ORDER BY c.some_date DESC` — sort the child rows; this is **inside** the subquery so it works per parent row
5. `FETCH FIRST 1 ROW ONLY` — take only the first (latest) child row for this parent
6. `) child_result` — alias the subquery result so outer `SELECT` can reference its columns
7. `child_result.child_col1` — reference the subquery output columns in the outer `SELECT`
8. If the subquery returns 0 rows for a parent row → that parent row is **dropped** (CROSS behaviour)

### `OUTER APPLY` Syntax

```sql
SELECT
    p.parent_id,
    p.parent_name,
    child_result.child_col1,   -- NULL when no matching child rows
    child_result.child_col2
FROM parent_table p
OUTER APPLY (
    SELECT c.child_col1, c.child_col2
    FROM child_table c
    WHERE c.parent_id = p.parent_id
    ORDER BY c.some_date DESC
    FETCH FIRST 1 ROW ONLY
) child_result
ORDER BY p.parent_id;
```

The only difference from `CROSS APPLY` is the `OUTER` keyword. When the subquery returns 0 rows, the parent row is **preserved** with NULLs in the `child_result` columns.

---

## 8. Database Setup: E-Commerce Schema

The following schema and data are used for every example in this guide. Run all statements in order.

### 8.1 Drop Existing Tables (Clean Slate)

```sql
-- Drop in dependency order: child tables first
BEGIN
    EXECUTE IMMEDIATE 'DROP TABLE audit_logs               CASCADE CONSTRAINTS PURGE';
EXCEPTION WHEN OTHERS THEN NULL; END;
/
BEGIN
    EXECUTE IMMEDIATE 'DROP TABLE customer_support_tickets CASCADE CONSTRAINTS PURGE';
EXCEPTION WHEN OTHERS THEN NULL; END;
/
BEGIN
    EXECUTE IMMEDIATE 'DROP TABLE payments                 CASCADE CONSTRAINTS PURGE';
EXCEPTION WHEN OTHERS THEN NULL; END;
/
BEGIN
    EXECUTE IMMEDIATE 'DROP TABLE order_items              CASCADE CONSTRAINTS PURGE';
EXCEPTION WHEN OTHERS THEN NULL; END;
/
BEGIN
    EXECUTE IMMEDIATE 'DROP TABLE orders                   CASCADE CONSTRAINTS PURGE';
EXCEPTION WHEN OTHERS THEN NULL; END;
/
BEGIN
    EXECUTE IMMEDIATE 'DROP TABLE products                 CASCADE CONSTRAINTS PURGE';
EXCEPTION WHEN OTHERS THEN NULL; END;
/
BEGIN
    EXECUTE IMMEDIATE 'DROP TABLE customers                CASCADE CONSTRAINTS PURGE';
EXCEPTION WHEN OTHERS THEN NULL; END;
/
```

### 8.2 Create Tables

```sql
-- ─────────────────────────────────────────────────────────────
-- CUSTOMERS
-- ─────────────────────────────────────────────────────────────
CREATE TABLE customers (
    customer_id  NUMBER         GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    full_name    VARCHAR2(100)  NOT NULL,
    email        VARCHAR2(150)  NOT NULL,
    phone        VARCHAR2(20),
    city         VARCHAR2(80),
    status       VARCHAR2(20)   DEFAULT 'ACTIVE'
                                CONSTRAINT chk_cust_status
                                CHECK (status IN ('ACTIVE', 'INACTIVE', 'BANNED')),
    created_at   DATE           DEFAULT SYSDATE NOT NULL,
    CONSTRAINT uq_customers_email UNIQUE (email)
);

-- ─────────────────────────────────────────────────────────────
-- PRODUCTS
-- ─────────────────────────────────────────────────────────────
CREATE TABLE products (
    product_id   NUMBER         GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    product_name VARCHAR2(150)  NOT NULL,
    category     VARCHAR2(80),
    unit_price   NUMBER(12, 2)  NOT NULL,
    stock_qty    NUMBER(10)     DEFAULT 0,
    created_at   DATE           DEFAULT SYSDATE NOT NULL,
    CONSTRAINT chk_product_price CHECK (unit_price >= 0)
);

-- ─────────────────────────────────────────────────────────────
-- ORDERS
-- ─────────────────────────────────────────────────────────────
CREATE TABLE orders (
    order_id     NUMBER         GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    customer_id  NUMBER         NOT NULL,
    order_date   DATE           DEFAULT SYSDATE NOT NULL,
    status       VARCHAR2(30)   DEFAULT 'PENDING'
                                CONSTRAINT chk_order_status
                                CHECK (status IN ('PENDING','CONFIRMED','SHIPPED','DELIVERED','CANCELLED')),
    total_amount NUMBER(14, 2)  DEFAULT 0,
    CONSTRAINT fk_orders_customer
        FOREIGN KEY (customer_id) REFERENCES customers (customer_id)
);

-- ─────────────────────────────────────────────────────────────
-- ORDER ITEMS
-- ─────────────────────────────────────────────────────────────
CREATE TABLE order_items (
    item_id      NUMBER         GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    order_id     NUMBER         NOT NULL,
    product_id   NUMBER         NOT NULL,
    quantity     NUMBER(8)      NOT NULL,
    unit_price   NUMBER(12, 2)  NOT NULL,
    line_amount  NUMBER(14, 2)  NOT NULL,
    CONSTRAINT chk_item_qty   CHECK (quantity > 0),
    CONSTRAINT chk_item_price CHECK (unit_price >= 0),
    CONSTRAINT fk_items_order
        FOREIGN KEY (order_id)   REFERENCES orders   (order_id),
    CONSTRAINT fk_items_product
        FOREIGN KEY (product_id) REFERENCES products (product_id)
);

-- ─────────────────────────────────────────────────────────────
-- PAYMENTS
-- ─────────────────────────────────────────────────────────────
CREATE TABLE payments (
    payment_id   NUMBER         GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    order_id     NUMBER         NOT NULL,
    payment_date DATE           DEFAULT SYSDATE NOT NULL,
    amount       NUMBER(14, 2)  NOT NULL,
    method       VARCHAR2(30)
                                CONSTRAINT chk_pay_method
                                CHECK (method IN ('CARD','BANK_TRANSFER','CASH','WALLET')),
    status       VARCHAR2(20)   DEFAULT 'PENDING'
                                CONSTRAINT chk_pay_status
                                CHECK (status IN ('PENDING','SUCCESS','FAILED','REFUNDED')),
    CONSTRAINT chk_pay_amount CHECK (amount > 0),
    CONSTRAINT fk_payments_order
        FOREIGN KEY (order_id) REFERENCES orders (order_id)
);

-- ─────────────────────────────────────────────────────────────
-- CUSTOMER SUPPORT TICKETS
-- ─────────────────────────────────────────────────────────────
CREATE TABLE customer_support_tickets (
    ticket_id    NUMBER         GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    customer_id  NUMBER         NOT NULL,
    order_id     NUMBER,                          -- nullable: ticket may not relate to an order
    subject      VARCHAR2(200)  NOT NULL,
    status       VARCHAR2(20)   DEFAULT 'OPEN'
                                CONSTRAINT chk_ticket_status
                                CHECK (status IN ('OPEN','IN_PROGRESS','RESOLVED','CLOSED')),
    priority     VARCHAR2(10)   DEFAULT 'MEDIUM'
                                CONSTRAINT chk_ticket_priority
                                CHECK (priority IN ('LOW','MEDIUM','HIGH','URGENT')),
    created_at   DATE           DEFAULT SYSDATE NOT NULL,
    resolved_at  DATE,
    CONSTRAINT fk_tickets_customer
        FOREIGN KEY (customer_id) REFERENCES customers (customer_id),
    CONSTRAINT fk_tickets_order
        FOREIGN KEY (order_id)    REFERENCES orders    (order_id)
);

-- ─────────────────────────────────────────────────────────────
-- AUDIT LOGS
-- ─────────────────────────────────────────────────────────────
CREATE TABLE audit_logs (
    log_id       NUMBER         GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    entity_type  VARCHAR2(50)   NOT NULL,
    entity_id    NUMBER         NOT NULL,
    action       VARCHAR2(30)   NOT NULL
                                CONSTRAINT chk_audit_action
                                CHECK (action IN ('CREATE','UPDATE','DELETE','VIEW')),
    changed_by   VARCHAR2(100),
    changed_at   DATE           DEFAULT SYSDATE NOT NULL,
    old_value    CLOB,
    new_value    CLOB
);
```

### 8.3 Sample Data

> **Data design notes:**
> - Alice and Bob have orders **and** successful payments
> - Carol has orders but **no payments**
> - Dave has orders but **no payments**
> - Eve has **no orders** but has a support ticket
> - Frank has **no orders and no tickets**
> - Product 7 (Webcam HD) has **never been sold**

```sql
-- ─── CUSTOMERS ───────────────────────────────────────────────
INSERT INTO customers (customer_id, full_name, email, phone, city, status, created_at)
VALUES (1, 'Alice Johnson',  'alice@example.com',  '+1-555-0101', 'New York',     'ACTIVE',   DATE '2023-10-01');

INSERT INTO customers (customer_id, full_name, email, phone, city, status, created_at)
VALUES (2, 'Bob Smith',      'bob@example.com',    '+1-555-0102', 'Los Angeles',  'ACTIVE',   DATE '2023-10-15');

INSERT INTO customers (customer_id, full_name, email, phone, city, status, created_at)
VALUES (3, 'Carol White',    'carol@example.com',  '+1-555-0103', 'Chicago',      'ACTIVE',   DATE '2023-11-05');

INSERT INTO customers (customer_id, full_name, email, phone, city, status, created_at)
VALUES (4, 'Dave Brown',     'dave@example.com',   '+1-555-0104', 'Houston',      'ACTIVE',   DATE '2023-11-20');

INSERT INTO customers (customer_id, full_name, email, phone, city, status, created_at)
VALUES (5, 'Eve Davis',      'eve@example.com',    '+1-555-0105', 'Phoenix',      'INACTIVE', DATE '2023-12-01');

INSERT INTO customers (customer_id, full_name, email, phone, city, status, created_at)
VALUES (6, 'Frank Miller',   'frank@example.com',  '+1-555-0106', 'Philadelphia', 'ACTIVE',   DATE '2024-01-10');

-- ─── PRODUCTS ────────────────────────────────────────────────
INSERT INTO products (product_id, product_name,         category,    unit_price, stock_qty)
VALUES (1, 'Laptop Pro 15',        'Electronics', 1299.99, 25);

INSERT INTO products (product_id, product_name,         category,    unit_price, stock_qty)
VALUES (2, 'Wireless Mouse',       'Electronics',   29.99, 150);

INSERT INTO products (product_id, product_name,         category,    unit_price, stock_qty)
VALUES (3, 'USB-C Hub',            'Electronics',   49.99,  80);

INSERT INTO products (product_id, product_name,         category,    unit_price, stock_qty)
VALUES (4, 'Desk Organizer',       'Office',         24.99, 200);

INSERT INTO products (product_id, product_name,         category,    unit_price, stock_qty)
VALUES (5, 'Ergonomic Chair',      'Furniture',     399.99,  15);

INSERT INTO products (product_id, product_name,         category,    unit_price, stock_qty)
VALUES (6, 'Mechanical Keyboard',  'Electronics',   149.99,  60);

INSERT INTO products (product_id, product_name,         category,    unit_price, stock_qty)
VALUES (7, 'Webcam HD 1080p',      'Electronics',    79.99,  40);  -- never sold

-- ─── ORDERS ──────────────────────────────────────────────────
-- Alice: two orders
INSERT INTO orders (order_id, customer_id, order_date, status, total_amount)
VALUES (1, 1, DATE '2024-01-10', 'DELIVERED', 1329.98);

INSERT INTO orders (order_id, customer_id, order_date, status, total_amount)
VALUES (2, 1, DATE '2024-03-22', 'SHIPPED',   199.98);

-- Bob: two orders
INSERT INTO orders (order_id, customer_id, order_date, status, total_amount)
VALUES (3, 2, DATE '2024-02-05', 'DELIVERED', 479.97);

INSERT INTO orders (order_id, customer_id, order_date, status, total_amount)
VALUES (4, 2, DATE '2024-04-18', 'CONFIRMED',  29.99);

-- Carol: one order (no payment)
INSERT INTO orders (order_id, customer_id, order_date, status, total_amount)
VALUES (5, 3, DATE '2024-03-01', 'PENDING',  1499.96);

-- Dave: one order (no payment)
INSERT INTO orders (order_id, customer_id, order_date, status, total_amount)
VALUES (6, 4, DATE '2024-02-20', 'CANCELLED',  79.97);

-- Eve and Frank: no orders

-- ─── ORDER ITEMS ─────────────────────────────────────────────
-- Order 1: Laptop + Wireless Mouse
INSERT INTO order_items (item_id, order_id, product_id, quantity, unit_price, line_amount)
VALUES (1, 1, 1, 1, 1299.99, 1299.99);  -- Laptop Pro 15

INSERT INTO order_items (item_id, order_id, product_id, quantity, unit_price, line_amount)
VALUES (2, 1, 2, 1,   29.99,   29.99);  -- Wireless Mouse

-- Order 2: Keyboard + USB-C Hub
INSERT INTO order_items (item_id, order_id, product_id, quantity, unit_price, line_amount)
VALUES (3, 2, 6, 1, 149.99, 149.99);    -- Mechanical Keyboard

INSERT INTO order_items (item_id, order_id, product_id, quantity, unit_price, line_amount)
VALUES (4, 2, 3, 1,  49.99,  49.99);    -- USB-C Hub

-- Order 3: Chair + USB-C Hub + Wireless Mouse
INSERT INTO order_items (item_id, order_id, product_id, quantity, unit_price, line_amount)
VALUES (5, 3, 5, 1, 399.99, 399.99);    -- Ergonomic Chair

INSERT INTO order_items (item_id, order_id, product_id, quantity, unit_price, line_amount)
VALUES (6, 3, 3, 1,  49.99,  49.99);    -- USB-C Hub

INSERT INTO order_items (item_id, order_id, product_id, quantity, unit_price, line_amount)
VALUES (7, 3, 2, 1,  29.99,  29.99);    -- Wireless Mouse

-- Order 4: Single mouse
INSERT INTO order_items (item_id, order_id, product_id, quantity, unit_price, line_amount)
VALUES (8, 4, 2, 1, 29.99, 29.99);      -- Wireless Mouse

-- Order 5: Laptop + 2x Desk Organizer + Keyboard
INSERT INTO order_items (item_id, order_id, product_id, quantity, unit_price, line_amount)
VALUES (9,  5, 1, 1, 1299.99, 1299.99); -- Laptop Pro 15

INSERT INTO order_items (item_id, order_id, product_id, quantity, unit_price, line_amount)
VALUES (10, 5, 4, 2,   24.99,   49.98); -- 2x Desk Organizer

INSERT INTO order_items (item_id, order_id, product_id, quantity, unit_price, line_amount)
VALUES (11, 5, 6, 1,  149.99,  149.99); -- Mechanical Keyboard

-- Order 6: 2x Desk Organizer + Wireless Mouse
INSERT INTO order_items (item_id, order_id, product_id, quantity, unit_price, line_amount)
VALUES (12, 6, 4, 2, 24.99,  49.98);    -- 2x Desk Organizer

INSERT INTO order_items (item_id, order_id, product_id, quantity, unit_price, line_amount)
VALUES (13, 6, 2, 1, 29.99,  29.99);    -- Wireless Mouse

-- ─── PAYMENTS ────────────────────────────────────────────────
-- Alice: both orders paid
INSERT INTO payments (payment_id, order_id, payment_date, amount, method, status)
VALUES (1, 1, DATE '2024-01-10', 1329.98, 'CARD',          'SUCCESS');

INSERT INTO payments (payment_id, order_id, payment_date, amount, method, status)
VALUES (2, 2, DATE '2024-03-22',  199.98, 'WALLET',        'SUCCESS');

-- Bob: order 3 paid, order 4 pending
INSERT INTO payments (payment_id, order_id, payment_date, amount, method, status)
VALUES (3, 3, DATE '2024-02-05',  479.97, 'BANK_TRANSFER', 'SUCCESS');

INSERT INTO payments (payment_id, order_id, payment_date, amount, method, status)
VALUES (4, 4, DATE '2024-04-18',   29.99, 'CARD',          'PENDING');

-- Carol and Dave: no payments (orders 5 and 6 are unpaid)

-- ─── SUPPORT TICKETS ─────────────────────────────────────────
-- Alice: 2 tickets
INSERT INTO customer_support_tickets
    (ticket_id, customer_id, order_id, subject, status, priority, created_at, resolved_at)
VALUES (1, 1, 1, 'Where is my order?',       'RESOLVED', 'HIGH',   DATE '2024-01-12', DATE '2024-01-14');

INSERT INTO customer_support_tickets
    (ticket_id, customer_id, order_id, subject, status, priority, created_at, resolved_at)
VALUES (2, 1, 2, 'Wrong item sent',           'OPEN',     'URGENT', DATE '2024-03-25', NULL);

-- Carol: 1 ticket
INSERT INTO customer_support_tickets
    (ticket_id, customer_id, order_id, subject, status, priority, created_at, resolved_at)
VALUES (3, 3, 5, 'Payment not processing',    'IN_PROGRESS', 'HIGH', DATE '2024-03-02', NULL);

-- Eve: 1 ticket (not related to an order)
INSERT INTO customer_support_tickets
    (ticket_id, customer_id, order_id, subject, status, priority, created_at, resolved_at)
VALUES (4, 5, NULL, 'Cannot log into account', 'OPEN',    'MEDIUM', DATE '2024-04-01', NULL);

-- Bob, Dave, Frank: no tickets

-- ─── AUDIT LOGS ──────────────────────────────────────────────
INSERT INTO audit_logs (log_id, entity_type, entity_id, action, changed_by, changed_at)
VALUES (1, 'order',    1, 'UPDATE', 'admin',  DATE '2024-01-15');

INSERT INTO audit_logs (log_id, entity_type, entity_id, action, changed_by, changed_at)
VALUES (2, 'customer', 5, 'UPDATE', 'admin',  DATE '2024-04-05');

INSERT INTO audit_logs (log_id, entity_type, entity_id, action, changed_by, changed_at)
VALUES (3, 'order',    3, 'VIEW',   'bob',    DATE '2024-02-06');

INSERT INTO audit_logs (log_id, entity_type, entity_id, action, changed_by, changed_at)
VALUES (4, 'payment',  1, 'CREATE', 'system', DATE '2024-01-10');

INSERT INTO audit_logs (log_id, entity_type, entity_id, action, changed_by, changed_at)
VALUES (5, 'order',    5, 'CREATE', 'carol',  DATE '2024-03-01');

COMMIT;
```

### 8.4 Customer Summary

| customer_id | full_name | Has Orders | Has Payments | Has Tickets |
|---|---|---|---|---|
| 1 | Alice Johnson | ✅ 2 orders | ✅ Both paid | ✅ 2 tickets |
| 2 | Bob Smith | ✅ 2 orders | ✅ 1 paid, 1 pending | ❌ None |
| 3 | Carol White | ✅ 1 order | ❌ Unpaid | ✅ 1 ticket |
| 4 | Dave Brown | ✅ 1 order | ❌ Unpaid | ❌ None |
| 5 | Eve Davis | ❌ None | ❌ N/A | ✅ 1 ticket |
| 6 | Frank Miller | ❌ None | ❌ N/A | ❌ None |

---

## 9. CROSS APPLY Examples

`CROSS APPLY` behaves like an inner join — it only returns left-side rows where the right-side subquery produces at least one result row.

---

### 9.1 Latest Order Per Customer

**Goal:** Return only customers who have placed at least one order. Show each customer's most recent order.

```sql
-- CROSS APPLY: latest order per customer
-- Only customers WITH orders are returned (inner join behaviour)
SELECT
    c.customer_id,
    c.full_name,
    c.city,
    latest.order_id,
    latest.order_date,
    latest.total_amount,
    latest.status       AS order_status
FROM customers c
CROSS APPLY (
    SELECT
        o.order_id,
        o.order_date,
        o.total_amount,
        o.status
    FROM orders o
    WHERE o.customer_id = c.customer_id   -- lateral reference
    ORDER BY o.order_date DESC             -- most recent first
    FETCH FIRST 1 ROW ONLY                -- take only the latest
) latest
ORDER BY c.customer_id;
```

**Expected output:**

| customer_id | full_name | city | order_id | order_date | total_amount | order_status |
|---|---|---|---|---|---|---|
| 1 | Alice Johnson | New York | 2 | 2024-03-22 | 199.98 | SHIPPED |
| 2 | Bob Smith | Los Angeles | 4 | 2024-04-18 | 29.99 | CONFIRMED |
| 3 | Carol White | Chicago | 5 | 2024-03-01 | 1499.96 | PENDING |
| 4 | Dave Brown | Houston | 6 | 2024-02-20 | 79.97 | CANCELLED |

> Eve (ID 5) and Frank (ID 6) are **not returned** because they have no orders.

---

### 9.2 Top 3 Products Per Order

**Goal:** For each order, return the top 3 line items by `line_amount` (highest value first).

```sql
-- CROSS APPLY: top 3 products by revenue for each order
SELECT
    o.order_id,
    TO_CHAR(o.order_date, 'YYYY-MM-DD') AS order_date,
    o.status                            AS order_status,
    top3.product_name,
    top3.quantity,
    top3.unit_price,
    top3.line_amount,
    top3.rank_in_order
FROM orders o
CROSS APPLY (
    SELECT
        p.product_name,
        oi.quantity,
        oi.unit_price,
        oi.line_amount,
        ROWNUM AS rank_in_order           -- rank within this order
    FROM order_items oi
    JOIN products p ON p.product_id = oi.product_id
    WHERE oi.order_id = o.order_id        -- lateral reference to current order
    ORDER BY oi.line_amount DESC          -- highest-value items first
    FETCH FIRST 3 ROWS ONLY              -- top 3 only
) top3
ORDER BY o.order_id, top3.rank_in_order;
```

**Expected output (partial):**

| order_id | order_date | product_name | quantity | unit_price | line_amount | rank_in_order |
|---|---|---|---|---|---|---|
| 1 | 2024-01-10 | Laptop Pro 15 | 1 | 1299.99 | 1299.99 | 1 |
| 1 | 2024-01-10 | Wireless Mouse | 1 | 29.99 | 29.99 | 2 |
| 3 | 2024-02-05 | Ergonomic Chair | 1 | 399.99 | 399.99 | 1 |
| 3 | 2024-02-05 | USB-C Hub | 1 | 49.99 | 49.99 | 2 |
| 3 | 2024-02-05 | Wireless Mouse | 1 | 29.99 | 29.99 | 3 |
| 5 | 2024-03-01 | Laptop Pro 15 | 1 | 1299.99 | 1299.99 | 1 |
| 5 | 2024-03-01 | Mechanical Keyboard | 1 | 149.99 | 149.99 | 2 |
| 5 | 2024-03-01 | Desk Organizer | 2 | 24.99 | 49.98 | 3 |

> Orders with more than 3 items (Order 3, Order 5) are correctly trimmed to 3.

---

### 9.3 Latest Successful Payment Per Order

**Goal:** Return only orders that have at least one `SUCCESS` payment. Show the most recent successful payment.

```sql
-- CROSS APPLY: only orders with a successful payment are returned
SELECT
    o.order_id,
    o.order_date,
    o.total_amount,
    o.status          AS order_status,
    paid.payment_id,
    paid.payment_date,
    paid.amount       AS paid_amount,
    paid.method
FROM orders o
CROSS APPLY (
    SELECT
        p.payment_id,
        p.payment_date,
        p.amount,
        p.method
    FROM payments p
    WHERE p.order_id = o.order_id         -- lateral reference
      AND p.status   = 'SUCCESS'          -- filter inside the subquery
    ORDER BY p.payment_date DESC
    FETCH FIRST 1 ROW ONLY
) paid
ORDER BY o.order_id;
```

**Expected output:**

| order_id | order_date | total_amount | order_status | payment_id | payment_date | paid_amount | method |
|---|---|---|---|---|---|---|---|
| 1 | 2024-01-10 | 1329.98 | DELIVERED | 1 | 2024-01-10 | 1329.98 | CARD |
| 2 | 2024-03-22 | 199.98 | SHIPPED | 2 | 2024-03-22 | 199.98 | WALLET |
| 3 | 2024-02-05 | 479.97 | DELIVERED | 3 | 2024-02-05 | 479.97 | BANK_TRANSFER |

> Orders 4 (PENDING payment), 5 and 6 (no payment) are **excluded**.

---

### 9.4 Customer's Most Recent Support Ticket

**Goal:** Return only customers who have raised at least one ticket. Show their most recent ticket.

```sql
-- CROSS APPLY: customers without tickets are excluded
SELECT
    c.customer_id,
    c.full_name,
    c.status         AS customer_status,
    t.ticket_id,
    t.subject,
    t.status         AS ticket_status,
    t.priority,
    t.created_at
FROM customers c
CROSS APPLY (
    SELECT
        cst.ticket_id,
        cst.subject,
        cst.status,
        cst.priority,
        cst.created_at
    FROM customer_support_tickets cst
    WHERE cst.customer_id = c.customer_id  -- lateral reference
    ORDER BY cst.created_at DESC
    FETCH FIRST 1 ROW ONLY
) t
ORDER BY c.customer_id;
```

**Expected output:**

| customer_id | full_name | customer_status | ticket_id | subject | ticket_status | priority | created_at |
|---|---|---|---|---|---|---|---|
| 1 | Alice Johnson | ACTIVE | 2 | Wrong item sent | OPEN | URGENT | 2024-03-25 |
| 3 | Carol White | ACTIVE | 3 | Payment not processing | IN_PROGRESS | HIGH | 2024-03-02 |
| 5 | Eve Davis | INACTIVE | 4 | Cannot log into account | OPEN | MEDIUM | 2024-04-01 |

> Bob (ID 2), Dave (ID 4), and Frank (ID 6) are **excluded** (no tickets).

---

### 9.5 Correlated Aggregation Per Customer

**Goal:** For each customer who has placed at least one order, calculate total orders, total spend, last order date, and average order amount in a single pass.

```sql
-- CROSS APPLY: multi-column aggregation per customer (only customers WITH orders)
-- HAVING COUNT(*) > 0 ensures no-order customers are excluded
SELECT
    c.customer_id,
    c.full_name,
    m.total_orders,
    m.total_spent,
    TO_CHAR(m.last_order_date, 'YYYY-MM-DD') AS last_order_date,
    m.avg_order_amount
FROM customers c
CROSS APPLY (
    SELECT
        COUNT(*)                           AS total_orders,
        SUM(o.total_amount)                AS total_spent,
        MAX(o.order_date)                  AS last_order_date,
        ROUND(AVG(o.total_amount), 2)      AS avg_order_amount
    FROM orders o
    WHERE o.customer_id = c.customer_id    -- lateral reference
    HAVING COUNT(*) > 0                    -- exclude no-order customers
) m
ORDER BY m.total_spent DESC;
```

**Expected output:**

| customer_id | full_name | total_orders | total_spent | last_order_date | avg_order_amount |
|---|---|---|---|---|---|
| 1 | Alice Johnson | 2 | 1529.96 | 2024-03-22 | 764.98 |
| 3 | Carol White | 1 | 1499.96 | 2024-03-01 | 1499.96 |
| 2 | Bob Smith | 2 | 509.96 | 2024-04-18 | 254.98 |
| 4 | Dave Brown | 1 | 79.97 | 2024-02-20 | 79.97 |

> **Why APPLY over four separate scalar subqueries?** Without APPLY, you would need four correlated scalar subqueries — hitting the `orders` table four times per customer. APPLY hits it once and returns all four columns together. See [Section 11.3](#113-apply-vs-correlated-subquery) for the comparison.

---

## 10. OUTER APPLY Examples

`OUTER APPLY` preserves **all left-side rows**, returning NULLs in the right-side columns when no match exists.

---

### 10.1 All Customers With Latest Order If Available

**Goal:** Show all six customers. For those with orders, show the latest one. For those without orders, show NULLs.

```sql
-- OUTER APPLY: all customers returned regardless of orders
SELECT
    c.customer_id,
    c.full_name,
    c.status          AS customer_status,
    lo.order_id,
    lo.order_date,
    lo.total_amount,
    lo.status         AS order_status,
    CASE
        WHEN lo.order_id IS NULL THEN 'No orders yet'
        ELSE 'Has orders'
    END               AS order_summary
FROM customers c
OUTER APPLY (
    SELECT
        o.order_id,
        o.order_date,
        o.total_amount,
        o.status
    FROM orders o
    WHERE o.customer_id = c.customer_id  -- lateral reference
    ORDER BY o.order_date DESC
    FETCH FIRST 1 ROW ONLY
) lo
ORDER BY c.customer_id;
```

**Expected output:**

| customer_id | full_name | customer_status | order_id | order_date | total_amount | order_status | order_summary |
|---|---|---|---|---|---|---|---|
| 1 | Alice Johnson | ACTIVE | 2 | 2024-03-22 | 199.98 | SHIPPED | Has orders |
| 2 | Bob Smith | ACTIVE | 4 | 2024-04-18 | 29.99 | CONFIRMED | Has orders |
| 3 | Carol White | ACTIVE | 5 | 2024-03-01 | 1499.96 | PENDING | Has orders |
| 4 | Dave Brown | ACTIVE | 6 | 2024-02-20 | 79.97 | CANCELLED | Has orders |
| 5 | Eve Davis | INACTIVE | NULL | NULL | NULL | NULL | No orders yet |
| 6 | Frank Miller | ACTIVE | NULL | NULL | NULL | NULL | No orders yet |

> All 6 customers appear. Eve and Frank have NULLs in order columns.

---

### 10.2 All Orders With Latest Payment If Available

**Goal:** Show all orders including unpaid ones. Show the most recent payment status where available.

```sql
-- OUTER APPLY: all orders returned, unpaid orders show NULL in payment columns
SELECT
    o.order_id,
    o.order_date,
    o.total_amount,
    o.status                                       AS order_status,
    lp.payment_id,
    lp.payment_date,
    lp.amount                                      AS payment_amount,
    lp.method,
    NVL(lp.status, 'UNPAID')                       AS payment_status
FROM orders o
OUTER APPLY (
    SELECT
        p.payment_id,
        p.payment_date,
        p.amount,
        p.method,
        p.status
    FROM payments p
    WHERE p.order_id = o.order_id                  -- lateral reference
    ORDER BY p.payment_date DESC
    FETCH FIRST 1 ROW ONLY
) lp
ORDER BY o.order_id;
```

**Expected output:**

| order_id | order_date | total_amount | order_status | payment_id | payment_date | payment_amount | method | payment_status |
|---|---|---|---|---|---|---|---|---|
| 1 | 2024-01-10 | 1329.98 | DELIVERED | 1 | 2024-01-10 | 1329.98 | CARD | SUCCESS |
| 2 | 2024-03-22 | 199.98 | SHIPPED | 2 | 2024-03-22 | 199.98 | WALLET | SUCCESS |
| 3 | 2024-02-05 | 479.97 | DELIVERED | 3 | 2024-02-05 | 479.97 | BANK_TRANSFER | SUCCESS |
| 4 | 2024-04-18 | 29.99 | CONFIRMED | 4 | 2024-04-18 | 29.99 | CARD | PENDING |
| 5 | 2024-03-01 | 1499.96 | PENDING | NULL | NULL | NULL | NULL | UNPAID |
| 6 | 2024-02-20 | 79.97 | CANCELLED | NULL | NULL | NULL | NULL | UNPAID |

> Orders 5 and 6 are included with `UNPAID` status because `NVL(lp.status, 'UNPAID')` handles the NULL.

---

### 10.3 All Customers With Last Support Ticket If Available

**Goal:** Show all customers. Include their most recent support ticket if one exists.

```sql
-- OUTER APPLY: all customers returned; ticketless customers show NULL
SELECT
    c.customer_id,
    c.full_name,
    t.ticket_id,
    t.subject,
    t.status         AS ticket_status,
    t.priority,
    t.created_at,
    CASE
        WHEN t.ticket_id IS NULL THEN 'No tickets'
        WHEN t.status IN ('OPEN', 'IN_PROGRESS') THEN 'Action required'
        ELSE 'Resolved'
    END              AS support_flag
FROM customers c
OUTER APPLY (
    SELECT
        cst.ticket_id,
        cst.subject,
        cst.status,
        cst.priority,
        cst.created_at
    FROM customer_support_tickets cst
    WHERE cst.customer_id = c.customer_id  -- lateral reference
    ORDER BY cst.created_at DESC
    FETCH FIRST 1 ROW ONLY
) t
ORDER BY c.customer_id;
```

**Expected output:**

| customer_id | full_name | ticket_id | subject | ticket_status | priority | created_at | support_flag |
|---|---|---|---|---|---|---|---|
| 1 | Alice Johnson | 2 | Wrong item sent | OPEN | URGENT | 2024-03-25 | Action required |
| 2 | Bob Smith | NULL | NULL | NULL | NULL | NULL | No tickets |
| 3 | Carol White | 3 | Payment not processing | IN_PROGRESS | HIGH | 2024-03-02 | Action required |
| 4 | Dave Brown | NULL | NULL | NULL | NULL | NULL | No tickets |
| 5 | Eve Davis | 4 | Cannot log into account | OPEN | MEDIUM | 2024-04-01 | Action required |
| 6 | Frank Miller | NULL | NULL | NULL | NULL | NULL | No tickets |

---

### 10.4 Customer Dashboard Query

**Goal:** Build a complete admin dashboard that shows every customer with their latest order, the payment status of that order, their most recent support ticket, aggregated order statistics, and a calculated risk status.

> **Advanced pattern:** Notice how the second `OUTER APPLY` (`lp`) references `lo.order_id` from the first. Each subsequent `APPLY` can reference columns from previous `APPLY` results in the same `FROM` clause, because Oracle processes them left to right.

```sql
-- OUTER APPLY: full customer dashboard (all customers, all columns)
SELECT
    -- Customer info
    c.customer_id,
    c.full_name,
    c.email,
    c.city,
    c.status                                   AS customer_status,

    -- Latest order (from first OUTER APPLY)
    lo.order_id                                AS latest_order_id,
    TO_CHAR(lo.order_date, 'YYYY-MM-DD')       AS latest_order_date,
    lo.total_amount                            AS latest_order_amount,
    lo.status                                  AS latest_order_status,

    -- Latest payment for that order (from second OUTER APPLY — references lo)
    TO_CHAR(lp.payment_date, 'YYYY-MM-DD')     AS latest_payment_date,
    NVL(lp.status, 'NO PAYMENT')              AS latest_payment_status,

    -- Latest support ticket (from third OUTER APPLY)
    lt.subject                                 AS latest_ticket_subject,
    lt.status                                  AS latest_ticket_status,
    lt.priority                                AS latest_ticket_priority,

    -- Aggregated stats (from fourth OUTER APPLY)
    NVL(os.total_orders, 0)                    AS total_orders,
    NVL(os.total_spent, 0)                     AS total_spent,

    -- Risk classification (business logic using all APPLY results)
    CASE
        WHEN os.total_orders IS NULL
             THEN 'NEW_CUSTOMER'
        WHEN lt.status IN ('OPEN', 'IN_PROGRESS')
             AND lt.priority IN ('HIGH', 'URGENT')
             THEN 'NEEDS_ATTENTION'
        WHEN lo.status = 'PENDING'
             AND (lp.payment_id IS NULL OR lp.status != 'SUCCESS')
             THEN 'PAYMENT_PENDING'
        WHEN c.status = 'INACTIVE'
             THEN 'INACTIVE_CUSTOMER'
        ELSE 'NORMAL'
    END                                        AS risk_status

FROM customers c

-- 1. Latest order per customer
OUTER APPLY (
    SELECT o.order_id, o.order_date, o.total_amount, o.status
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC
    FETCH FIRST 1 ROW ONLY
) lo

-- 2. Latest payment for the latest order (references lo.order_id!)
OUTER APPLY (
    SELECT p.payment_id, p.payment_date, p.amount, p.status
    FROM payments p
    WHERE p.order_id = lo.order_id            -- references previous APPLY result
    ORDER BY p.payment_date DESC
    FETCH FIRST 1 ROW ONLY
) lp

-- 3. Latest support ticket per customer
OUTER APPLY (
    SELECT cst.ticket_id, cst.subject, cst.status, cst.priority
    FROM customer_support_tickets cst
    WHERE cst.customer_id = c.customer_id
    ORDER BY cst.created_at DESC
    FETCH FIRST 1 ROW ONLY
) lt

-- 4. Aggregated order statistics
OUTER APPLY (
    SELECT
        COUNT(*)              AS total_orders,
        SUM(o2.total_amount)  AS total_spent
    FROM orders o2
    WHERE o2.customer_id = c.customer_id
) os

ORDER BY c.customer_id;
```

**Expected output (key columns):**

| customer_id | full_name | latest_order_status | latest_payment_status | latest_ticket_status | total_orders | total_spent | risk_status |
|---|---|---|---|---|---|---|---|
| 1 | Alice Johnson | SHIPPED | SUCCESS | OPEN | 2 | 1529.96 | NEEDS_ATTENTION |
| 2 | Bob Smith | CONFIRMED | PENDING | NO TICKET | 2 | 509.96 | NORMAL |
| 3 | Carol White | PENDING | NO PAYMENT | IN_PROGRESS | 1 | 1499.96 | NEEDS_ATTENTION |
| 4 | Dave Brown | CANCELLED | NO PAYMENT | NO TICKET | 1 | 79.97 | NORMAL |
| 5 | Eve Davis | NULL | NO PAYMENT | OPEN | 0 | 0 | NEW_CUSTOMER |
| 6 | Frank Miller | NULL | NO PAYMENT | NO TICKET | 0 | 0 | NEW_CUSTOMER |

> This single query replaces what would otherwise be 4+ separate queries or multiple correlated subqueries. It is the canonical example of why `OUTER APPLY` is invaluable for reporting.

---

### 10.5 Product Sales Summary

**Goal:** Show all products, including the Webcam HD that has never been ordered. Show sales statistics for each.

```sql
-- OUTER APPLY: all products including unsold ones
SELECT
    p.product_id,
    p.product_name,
    p.category,
    p.unit_price,
    p.stock_qty,
    NVL(ps.units_sold,   0)        AS units_sold,
    NVL(ps.total_revenue, 0)       AS total_revenue,
    NVL(ps.order_count,  0)        AS order_count,
    TO_CHAR(ps.last_sold_date, 'YYYY-MM-DD') AS last_sold_date,
    CASE
        WHEN ps.order_count IS NULL THEN 'NEVER SOLD'
        WHEN ps.last_sold_date < DATE '2024-02-01' THEN 'SLOW MOVING'
        ELSE 'ACTIVE'
    END                            AS sales_status
FROM products p
OUTER APPLY (
    SELECT
        SUM(oi.quantity)            AS units_sold,
        SUM(oi.line_amount)         AS total_revenue,
        COUNT(DISTINCT oi.order_id) AS order_count,
        MAX(o.order_date)           AS last_sold_date
    FROM order_items oi
    JOIN orders o ON o.order_id = oi.order_id
    WHERE oi.product_id = p.product_id    -- lateral reference
) ps
ORDER BY NVL(ps.total_revenue, 0) DESC;
```

**Expected output:**

| product_id | product_name | unit_price | units_sold | total_revenue | order_count | last_sold_date | sales_status |
|---|---|---|---|---|---|---|---|
| 1 | Laptop Pro 15 | 1299.99 | 2 | 2599.98 | 2 | 2024-03-01 | ACTIVE |
| 5 | Ergonomic Chair | 399.99 | 1 | 399.99 | 1 | 2024-02-05 | SLOW MOVING |
| 6 | Mechanical Keyboard | 149.99 | 2 | 299.98 | 2 | 2024-03-01 | ACTIVE |
| 2 | Wireless Mouse | 29.99 | 4 | 119.96 | 4 | 2024-04-18 | ACTIVE |
| 3 | USB-C Hub | 49.99 | 2 | 99.98 | 2 | 2024-03-22 | ACTIVE |
| 4 | Desk Organizer | 24.99 | 6 | 149.88 | 2 | 2024-03-01 | ACTIVE |
| 7 | Webcam HD 1080p | 79.99 | 0 | 0 | 0 | NULL | NEVER SOLD |

> Product 7 (Webcam HD) appears with all zeros because `OUTER APPLY` preserves it even though the aggregation returns no rows.

---

## 11. APPLY vs Other SQL Techniques

### 11.1 CROSS APPLY vs INNER JOIN

A standard `INNER JOIN` cannot include `ORDER BY` or `FETCH FIRST` inside the join condition. For flat relationships (no top-N needed), both produce the same result. For correlated top-N, only `CROSS APPLY` works directly.

**When they are equivalent (no top-N needed):**

```sql
-- INNER JOIN: returns all orders for customers who have any orders
SELECT c.full_name, o.order_id, o.total_amount
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id;

-- CROSS APPLY equivalent (no FETCH FIRST = same as join)
SELECT c.full_name, o.order_id, o.total_amount
FROM customers c
CROSS APPLY (
    SELECT order_id, total_amount
    FROM orders o
    WHERE o.customer_id = c.customer_id
) o;
```

**When CROSS APPLY is essential (top-N per group):**

```sql
-- This is impossible with a standard INNER JOIN:
SELECT c.full_name, latest.order_id, latest.order_date
FROM customers c
CROSS APPLY (
    SELECT order_id, order_date
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC
    FETCH FIRST 1 ROW ONLY              -- impossible in a JOIN condition
) latest;
```

**Rule:** Use `INNER JOIN` for flat, unrestricted joins. Use `CROSS APPLY` when the right side needs `ORDER BY`, `FETCH FIRST`, or complex correlated logic.

---

### 11.2 OUTER APPLY vs LEFT JOIN

Similar to `INNER JOIN`, a `LEFT JOIN` cannot contain `ORDER BY` or `FETCH FIRST`. For simple relationships without row limiting, they are equivalent.

**When they are equivalent:**

```sql
-- LEFT JOIN: all customers, NULLs for those without orders
SELECT c.full_name, o.order_id, o.total_amount
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id;
-- Returns MULTIPLE rows per customer if they have multiple orders

-- OUTER APPLY: also returns multiple rows per customer
SELECT c.full_name, o.order_id, o.total_amount
FROM customers c
OUTER APPLY (
    SELECT order_id, total_amount
    FROM orders o
    WHERE o.customer_id = c.customer_id
) o;
```

**When OUTER APPLY is essential (top-N with nulls preserved):**

```sql
-- OUTER APPLY: all customers; only 1 order row per customer (the latest)
SELECT c.full_name, lo.order_id, lo.order_date
FROM customers c
OUTER APPLY (
    SELECT order_id, order_date
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC
    FETCH FIRST 1 ROW ONLY             -- LEFT JOIN cannot do this
) lo;
```

**Rule:** Use `LEFT JOIN` for unrestricted left outer joins. Use `OUTER APPLY` when you need ordered, row-limited, or aggregated correlated results while preserving all left-side rows.

---

### 11.3 APPLY vs Correlated Subquery

A correlated subquery in the `SELECT` list can return only **one column and one value**. To return multiple calculated columns, you must write multiple subqueries — each hitting the same table again.

**Correlated subqueries — verbose, multiple table hits:**

```sql
SELECT
    c.customer_id,
    c.full_name,
    (SELECT COUNT(*)          FROM orders o WHERE o.customer_id = c.customer_id) AS total_orders,
    (SELECT SUM(total_amount) FROM orders o WHERE o.customer_id = c.customer_id) AS total_spent,
    (SELECT MAX(order_date)   FROM orders o WHERE o.customer_id = c.customer_id) AS last_order_date,
    (SELECT AVG(total_amount) FROM orders o WHERE o.customer_id = c.customer_id) AS avg_order
FROM customers c;
-- Hits the orders table FOUR times per customer row
```

**CROSS APPLY — concise, single table hit per customer:**

```sql
SELECT
    c.customer_id,
    c.full_name,
    m.total_orders,
    m.total_spent,
    m.last_order_date,
    m.avg_order
FROM customers c
CROSS APPLY (
    SELECT
        COUNT(*)                       AS total_orders,
        SUM(o.total_amount)            AS total_spent,
        MAX(o.order_date)              AS last_order_date,
        ROUND(AVG(o.total_amount), 2)  AS avg_order
    FROM orders o
    WHERE o.customer_id = c.customer_id
    HAVING COUNT(*) > 0
) m;
-- Hits orders ONCE per customer row — cleaner and faster
```

**Rule:** When you need more than one column from a correlated computation, `APPLY` is almost always cleaner and performs better than multiple scalar subqueries.

---

### 11.4 APPLY vs Window Functions

`ROW_NUMBER() OVER (PARTITION BY ...)` is the classic alternative to `CROSS APPLY` for top-N per group.

**Using `CROSS APPLY` (or `OUTER APPLY`):**

```sql
-- CROSS APPLY: latest order per customer (only customers with orders)
SELECT c.customer_id, c.full_name, lo.order_id, lo.order_date, lo.total_amount
FROM customers c
CROSS APPLY (
    SELECT o.order_id, o.order_date, o.total_amount
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC
    FETCH FIRST 1 ROW ONLY
) lo;
```

**Using `ROW_NUMBER()` window function:**

```sql
-- ROW_NUMBER: equivalent result using window function
SELECT customer_id, full_name, order_id, order_date, total_amount
FROM (
    SELECT
        c.customer_id,
        c.full_name,
        o.order_id,
        o.order_date,
        o.total_amount,
        ROW_NUMBER() OVER (
            PARTITION BY c.customer_id
            ORDER BY o.order_date DESC
        ) AS rn
    FROM customers c
    JOIN orders o ON o.customer_id = c.customer_id
)
WHERE rn = 1;
```

**Comparison:**

| Aspect | CROSS APPLY | ROW_NUMBER() |
|---|---|---|
| Readability | Very clear intent | Requires a wrapping subquery |
| Performance (small N) | Often faster — uses index, fetches 1 row early | Scans full join, then filters |
| Performance (large N) | Better for small N per group with good index | Can be better for large N with parallel scan |
| OUTER behaviour | Use `OUTER APPLY` | Add `LEFT JOIN` + handle NULLs manually |
| Complex right-side logic | Handled inline | Gets very complex |
| Supported in Oracle 11g | No | Yes |

**Rule:**
- For top-1 or top-N queries with a good index on `(customer_id, order_date)`, `CROSS APPLY` / `OUTER APPLY` is usually faster and cleaner.
- For large full-table aggregations or when you need multiple window calculations in one pass, `ROW_NUMBER()` (or other window functions) is often more efficient.
- Always verify with `EXPLAIN PLAN` on your actual data.

---

## 12. LATERAL Inline Views in Oracle

### What Is `LATERAL`?

`LATERAL` is the SQL standard keyword that enables an inline view (a subquery in the `FROM` clause) to reference columns from tables listed to its left. Oracle introduced `LATERAL` in version 12c R1 (before `CROSS APPLY` and `OUTER APPLY`).

`CROSS APPLY` and `OUTER APPLY` are essentially Oracle aliases for specific `LATERAL` join patterns:

```
CROSS APPLY (...)   ≡   CROSS JOIN LATERAL (...)
OUTER APPLY (...)   ≡   LEFT JOIN LATERAL (...) ON 1=1
```

### Syntax Comparison

**`CROSS APPLY` and its `LATERAL` equivalent:**

```sql
-- Using CROSS APPLY
SELECT c.customer_id, c.full_name, lo.order_id, lo.order_date
FROM customers c
CROSS APPLY (
    SELECT o.order_id, o.order_date
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC
    FETCH FIRST 1 ROW ONLY
) lo;

-- Equivalent using CROSS JOIN LATERAL
SELECT c.customer_id, c.full_name, lo.order_id, lo.order_date
FROM customers c
CROSS JOIN LATERAL (
    SELECT o.order_id, o.order_date
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC
    FETCH FIRST 1 ROW ONLY
) lo;

-- Equivalent using comma syntax with LATERAL (also CROSS JOIN semantics)
SELECT c.customer_id, c.full_name, lo.order_id, lo.order_date
FROM customers c,
LATERAL (
    SELECT o.order_id, o.order_date
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC
    FETCH FIRST 1 ROW ONLY
) lo;
```

**`OUTER APPLY` and its `LATERAL` equivalent:**

```sql
-- Using OUTER APPLY
SELECT c.customer_id, c.full_name, lo.order_id, lo.order_date
FROM customers c
OUTER APPLY (
    SELECT o.order_id, o.order_date
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC
    FETCH FIRST 1 ROW ONLY
) lo;

-- Equivalent using LEFT JOIN LATERAL
SELECT c.customer_id, c.full_name, lo.order_id, lo.order_date
FROM customers c
LEFT JOIN LATERAL (
    SELECT o.order_id, o.order_date
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC
    FETCH FIRST 1 ROW ONLY
) lo ON 1 = 1;  -- Oracle requires ON clause for LATERAL LEFT JOIN
```

### Readability and Portability

| Syntax | Oracle | SQL Server | PostgreSQL | Readability |
|---|---|---|---|---|
| `CROSS APPLY` | 12c R2+ | 2005+ | ❌ | Explicit inner semantics |
| `OUTER APPLY` | 12c R2+ | 2005+ | ❌ | Explicit outer semantics |
| `CROSS JOIN LATERAL` | 12c R1+ | ❌ | ✅ | Verbose but portable |
| `LEFT JOIN LATERAL ON 1=1` | 12c R1+ | ❌ | ✅ | Verbose |
| `, LATERAL (...)` | 12c R1+ | ❌ | ✅ | Concise but implicit join type |

**Recommendation for Oracle-only codebases:** Use `CROSS APPLY` and `OUTER APPLY`. They are the most expressive and clearly communicate intent.

**Recommendation for cross-database portability:** Use `CROSS JOIN LATERAL` and `LEFT JOIN LATERAL ... ON 1=1` for code that must run on both Oracle and PostgreSQL.

---

## 13. Advanced Business Scenario: Admin Dashboard

### The Problem

An e-commerce company's admin team needs a single endpoint that returns a complete customer overview for a reporting dashboard. Requirements:

- Every customer must appear, even those who have never ordered
- Show the customer's latest order date and amount
- Show the payment status of that latest order
- Show the most recent support ticket
- Show total lifetime spend and order count
- Assign a risk label so the support team can prioritize

### Why Standard Joins Fail Here

```sql
-- ATTEMPT with LEFT JOINs: breaks down because of multiple-row joins
SELECT c.full_name, o.order_date, p.status, t.subject
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id   -- returns ALL orders, not latest
LEFT JOIN payments p ON p.order_id = o.order_id        -- ambiguous: which order's payment?
LEFT JOIN customer_support_tickets t ON t.customer_id = c.customer_id; -- multiple tickets per customer
-- Result: a cartesian-like explosion of rows; every order × every payment × every ticket
```

This produces incorrect row counts and requires complicated deduplication. `OUTER APPLY` solves it elegantly because each subquery returns exactly 1 row (or NULL) per customer:

```sql
-- OUTER APPLY admin dashboard: correct, clean, single-row-per-customer
SELECT
    c.customer_id,
    c.full_name,
    c.email,
    c.status                                    AS account_status,

    -- Latest order
    NVL(TO_CHAR(lo.order_date, 'YYYY-MM-DD'),
        'No orders')                            AS last_order_date,
    NVL(TO_CHAR(lo.total_amount, 'FM99999990.00'),
        '-')                                    AS last_order_amount,
    NVL(lo.status, '-')                         AS last_order_status,

    -- Payment on latest order
    NVL(lp.status, 'UNPAID')                    AS last_payment_status,

    -- Latest support ticket
    NVL(lt.subject, 'No open tickets')          AS last_ticket_subject,
    NVL(lt.status, '-')                         AS last_ticket_status,
    NVL(lt.priority, '-')                       AS last_ticket_priority,

    -- Lifetime stats
    NVL(os.total_orders, 0)                     AS total_orders,
    NVL(os.lifetime_spend, 0)                   AS lifetime_spend,

    -- Risk label
    CASE
        WHEN os.total_orders IS NULL
             THEN 'NEW'
        WHEN lt.status IN ('OPEN','IN_PROGRESS')
             AND lt.priority IN ('HIGH','URGENT')
             THEN 'ESCALATED'
        WHEN lo.status IN ('PENDING','CONFIRMED')
             AND NVL(lp.status,'UNPAID') NOT IN ('SUCCESS')
             THEN 'PAYMENT_AT_RISK'
        WHEN c.status = 'INACTIVE'
             THEN 'CHURNED'
        ELSE 'HEALTHY'
    END                                         AS risk_label

FROM customers c

OUTER APPLY (
    SELECT o.order_id, o.order_date, o.total_amount, o.status
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC
    FETCH FIRST 1 ROW ONLY
) lo

OUTER APPLY (
    SELECT p.payment_id, p.status
    FROM payments p
    WHERE p.order_id = lo.order_id
    ORDER BY p.payment_date DESC
    FETCH FIRST 1 ROW ONLY
) lp

OUTER APPLY (
    SELECT cst.ticket_id, cst.subject, cst.status, cst.priority
    FROM customer_support_tickets cst
    WHERE cst.customer_id = c.customer_id
      AND cst.status IN ('OPEN', 'IN_PROGRESS')   -- only active tickets
    ORDER BY cst.created_at DESC
    FETCH FIRST 1 ROW ONLY
) lt

OUTER APPLY (
    SELECT COUNT(*) AS total_orders, SUM(o2.total_amount) AS lifetime_spend
    FROM orders o2
    WHERE o2.customer_id = c.customer_id
) os

ORDER BY
    CASE risk_label
        WHEN 'ESCALATED'      THEN 1
        WHEN 'PAYMENT_AT_RISK' THEN 2
        WHEN 'CHURNED'        THEN 3
        WHEN 'NEW'            THEN 4
        ELSE 5
    END,
    os.lifetime_spend DESC NULLS LAST;
```

This single query returns exactly one row per customer, sorted by risk priority, with no duplicates and no post-processing needed.

---

## 14. Performance Tuning

### How Oracle Executes APPLY Queries

Oracle typically executes `APPLY` queries using a **Nested Loops** strategy:

1. Oracle scans the left-side table (outer loop)
2. For each row, it executes the right-side subquery (inner loop)
3. With a proper index on the correlated column, the inner loop is an **index range scan** — extremely fast (O(log n) per outer row)
4. Without an index, the inner loop is a **full table scan** — O(n) per outer row, making the whole query O(n×m) — catastrophic at scale

### Recommended Indexes

```sql
-- Index for latest order per customer
-- Covers: WHERE customer_id = ? ORDER BY order_date DESC FETCH FIRST 1 ROW ONLY
CREATE INDEX idx_orders_cust_date
    ON orders (customer_id, order_date DESC);

-- Index for payments per order
CREATE INDEX idx_payments_order_date
    ON payments (order_id, payment_date DESC);

-- Index for top-N products per order
CREATE INDEX idx_items_order_amount
    ON order_items (order_id, line_amount DESC);

-- Index for support tickets per customer
CREATE INDEX idx_tickets_cust_date
    ON customer_support_tickets (customer_id, created_at DESC);

-- Additional partial index for active tickets only
CREATE INDEX idx_tickets_cust_active
    ON customer_support_tickets (customer_id, created_at DESC)
    WHERE status IN ('OPEN', 'IN_PROGRESS');  -- Oracle 12c+ function-based/partial index
```

> **Why `(customer_id, order_date DESC)` and not just `(customer_id)`?** With `ORDER BY order_date DESC FETCH FIRST 1 ROW ONLY`, Oracle can satisfy the entire inner query with a single index entry lookup — no sort step required. A plain `(customer_id)` index still requires a sort of matching rows, which is much slower.

### Inspecting the Execution Plan

```sql
-- Step 1: Generate the plan
EXPLAIN PLAN FOR
SELECT c.customer_id, c.full_name, lo.order_id, lo.order_date
FROM customers c
OUTER APPLY (
    SELECT o.order_id, o.order_date
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC
    FETCH FIRST 1 ROW ONLY
) lo;

-- Step 2: Display the plan
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

**Good plan indicators:**
- `NESTED LOOPS` at the top
- `INDEX RANGE SCAN` on `idx_orders_cust_date` for the inner query
- Row count of 1 returned by the inner loop (`Rows=1`)
- No `SORT ORDER BY` operation (the index handles ordering)

**Bad plan indicators:**
- `HASH JOIN` or `MERGE JOIN` (optimizer chose not to use nested loops)
- `TABLE ACCESS FULL` on the inner table (missing index)
- `SORT ORDER BY` followed by `COUNT STOPKEY` (no index for ordering)
- Very high estimated cardinality mismatch

### `FETCH FIRST` Optimisation

```sql
-- Oracle internally transforms FETCH FIRST 1 ROW ONLY into a COUNT STOPKEY:
-- SELECT ... WHERE ROWNUM = 1 ORDER BY ...
-- With a covering index, Oracle stops reading after finding the first matching row.
-- Without an index, it must read ALL matching rows, sort them, and THEN stop.
-- The difference can be seconds vs milliseconds for large tables.
```

### Common Reasons APPLY Queries Become Slow

- **No index on the correlated column** — Oracle scans the full inner table for each outer row
- **Index exists but does not include the ORDER BY column** — sort step forces full scan of matching rows before `FETCH FIRST` kicks in
- **Returning too many rows from the right side** — using `FETCH FIRST 100 ROWS ONLY` when `FETCH FIRST 1 ROW ONLY` is enough
- **Filtering too late** — putting conditions in the outer `WHERE` that should be inside the `APPLY` subquery
- **Calling expensive functions inside the subquery** — `APPLY` calls the subquery once per outer row; expensive functions multiply
- **Unnecessary `APPLY` for a simple join** — use `INNER JOIN` when you don't need top-N or correlated row limits
- **Missing `ORDER BY` with `FETCH FIRST`** — without deterministic ordering, the "top" row is random and the query is logically incorrect

### Gathering Optimizer Statistics

```sql
-- After loading sample data, gather table statistics for better plans
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS(
        ownname   => USER,
        tabname   => 'ORDERS',
        cascade   => TRUE        -- includes indexes
    );
END;
/
```

---

## 15. Laravel Developer Guide

### Why Laravel Developers Need to Understand APPLY

Laravel's Eloquent ORM and Query Builder do not support `CROSS APPLY`, `OUTER APPLY`, or correlated `FETCH FIRST` queries. When your application runs on Oracle and needs:

- Latest order per customer on a dashboard page
- Top 3 products per order in a sales report
- Per-customer metrics in an API response
- Any query with ordered, row-limited correlated subqueries

You must use raw SQL via `DB::select()`. Understanding `APPLY` lets you replace N+1 query loops with a single, efficient database round trip.

### Using `DB::select()` with OUTER APPLY

```php
<?php

use Illuminate\Support\Facades\DB;

// Basic OUTER APPLY query using named bind parameters (Oracle style: :param)
$customers = DB::select("
    SELECT
        c.customer_id,
        c.full_name,
        c.email,
        lo.order_id         AS latest_order_id,
        lo.order_date       AS latest_order_date,
        lo.total_amount     AS latest_order_amount,
        lo.status           AS latest_order_status,
        NVL(lp.status, 'UNPAID') AS payment_status,
        NVL(os.total_orders, 0)  AS total_orders,
        NVL(os.total_spent,  0)  AS total_spent
    FROM customers c
    OUTER APPLY (
        SELECT o.order_id, o.order_date, o.total_amount, o.status
        FROM orders o
        WHERE o.customer_id = c.customer_id
        ORDER BY o.order_date DESC
        FETCH FIRST 1 ROW ONLY
    ) lo
    OUTER APPLY (
        SELECT p.status
        FROM payments p
        WHERE p.order_id = lo.order_id
        ORDER BY p.payment_date DESC
        FETCH FIRST 1 ROW ONLY
    ) lp
    OUTER APPLY (
        SELECT COUNT(*) AS total_orders, SUM(o2.total_amount) AS total_spent
        FROM orders o2
        WHERE o2.customer_id = c.customer_id
    ) os
    WHERE c.status = :status
    ORDER BY c.customer_id
", ['status' => 'ACTIVE']);
```

### Using Bind Parameters

```php
<?php

// Always use named bind parameters — never concatenate user input into SQL
$results = DB::select("
    SELECT
        c.customer_id,
        c.full_name,
        latest.order_id,
        latest.order_date,
        latest.total_amount
    FROM customers c
    CROSS APPLY (
        SELECT o.order_id, o.order_date, o.total_amount
        FROM orders o
        WHERE o.customer_id = c.customer_id
          AND o.order_date  >= :start_date    -- named bind parameter
          AND o.order_date  <= :end_date
        ORDER BY o.order_date DESC
        FETCH FIRST 1 ROW ONLY
    ) latest
    WHERE c.status = :status
    ORDER BY c.customer_id
", [
    'status'     => 'ACTIVE',
    'start_date' => '2024-01-01',
    'end_date'   => '2024-12-31',
]);
```

### Service Class Example

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\DB;
use Illuminate\Support\Collection;

class CustomerDashboardService
{
    /**
     * Get the full dashboard view for a single customer.
     * Uses OUTER APPLY to fetch all data in one query.
     */
    public function getCustomerDashboard(int $customerId): array
    {
        $rows = DB::select("
            SELECT
                c.customer_id,
                c.full_name,
                c.email,
                c.status                                AS account_status,
                lo.order_id                             AS latest_order_id,
                TO_CHAR(lo.order_date, 'YYYY-MM-DD')    AS latest_order_date,
                lo.total_amount                         AS latest_order_amount,
                NVL(lo.status, 'NO ORDERS')             AS latest_order_status,
                NVL(lp.status, 'UNPAID')                AS payment_status,
                lt.subject                              AS latest_ticket,
                lt.status                               AS ticket_status,
                lt.priority                             AS ticket_priority,
                NVL(os.total_orders, 0)                 AS total_orders,
                NVL(os.lifetime_spend, 0)               AS lifetime_spend
            FROM customers c
            OUTER APPLY (
                SELECT o.order_id, o.order_date, o.total_amount, o.status
                FROM orders o
                WHERE o.customer_id = c.customer_id
                ORDER BY o.order_date DESC
                FETCH FIRST 1 ROW ONLY
            ) lo
            OUTER APPLY (
                SELECT p.status
                FROM payments p
                WHERE p.order_id = lo.order_id
                ORDER BY p.payment_date DESC
                FETCH FIRST 1 ROW ONLY
            ) lp
            OUTER APPLY (
                SELECT cst.subject, cst.status, cst.priority
                FROM customer_support_tickets cst
                WHERE cst.customer_id = c.customer_id
                ORDER BY cst.created_at DESC
                FETCH FIRST 1 ROW ONLY
            ) lt
            OUTER APPLY (
                SELECT COUNT(*) AS total_orders, SUM(o2.total_amount) AS lifetime_spend
                FROM orders o2
                WHERE o2.customer_id = c.customer_id
            ) os
            WHERE c.customer_id = :customer_id
        ", ['customer_id' => $customerId]);

        if (empty($rows)) {
            return [];
        }

        return (array) $rows[0];
    }

    /**
     * Get latest order summary for multiple customers at once.
     * Replaces N+1 queries with a single CROSS APPLY call.
     */
    public function getLatestOrdersForCustomers(array $customerIds): Collection
    {
        if (empty($customerIds)) {
            return collect();
        }

        // Build the IN list safely using named bind parameters
        $bindings = [];
        $placeholders = [];

        foreach ($customerIds as $index => $id) {
            $key = 'cid_' . $index;
            $placeholders[] = ':' . $key;
            $bindings[$key] = $id;
        }

        $inClause = implode(', ', $placeholders);

        $rows = DB::select("
            SELECT
                c.customer_id,
                c.full_name,
                latest.order_id,
                latest.order_date,
                latest.total_amount,
                latest.status AS order_status
            FROM customers c
            CROSS APPLY (
                SELECT o.order_id, o.order_date, o.total_amount, o.status
                FROM orders o
                WHERE o.customer_id = c.customer_id
                ORDER BY o.order_date DESC
                FETCH FIRST 1 ROW ONLY
            ) latest
            WHERE c.customer_id IN ($inClause)
            ORDER BY c.customer_id
        ", $bindings);

        return collect($rows)->keyBy('customer_id');
    }

    /**
     * Get top-N products per order for an order list page.
     */
    public function getTopProductsPerOrder(array $orderIds, int $topN = 3): Collection
    {
        if (empty($orderIds)) {
            return collect();
        }

        $bindings = ['top_n' => $topN];
        $placeholders = [];

        foreach ($orderIds as $index => $id) {
            $key = 'oid_' . $index;
            $placeholders[] = ':' . $key;
            $bindings[$key] = $id;
        }

        $inClause = implode(', ', $placeholders);

        $rows = DB::select("
            SELECT
                o.order_id,
                top_items.product_name,
                top_items.quantity,
                top_items.line_amount
            FROM orders o
            CROSS APPLY (
                SELECT p.product_name, oi.quantity, oi.line_amount
                FROM order_items oi
                JOIN products p ON p.product_id = oi.product_id
                WHERE oi.order_id = o.order_id
                ORDER BY oi.line_amount DESC
                FETCH FIRST :top_n ROWS ONLY
            ) top_items
            WHERE o.order_id IN ($inClause)
            ORDER BY o.order_id, top_items.line_amount DESC
        ", $bindings);

        // Group by order_id for easy template rendering
        return collect($rows)->groupBy('order_id');
    }
}
```

### Why Raw SQL Is Better Than Eloquent for APPLY Queries

Eloquent's Query Builder builds SQL from method chains. It has no concept of `CROSS APPLY`, `OUTER APPLY`, or correlated `ORDER BY` inside subqueries. Attempting to simulate this with Eloquent leads to:

- Multiple database round trips (N+1 pattern)
- Large result sets loaded into PHP memory then filtered in PHP
- Complex, unreadable relationship chains that produce wrong SQL

Raw SQL with `DB::select()` gives you full control, full Oracle feature access, and better performance.

### How APPLY Eliminates N+1 Problems

```php
// ❌ N+1 PROBLEM: one query per customer
$customers = Customer::all(); // 1 query
foreach ($customers as $customer) {
    $customer->latestOrder = Order::where('customer_id', $customer->id)
        ->orderByDesc('order_date')
        ->first(); // 1 query per customer = N queries total
}
// 1 + N database round trips

// ✅ APPLY SOLUTION: one query for all customers
$customers = DB::select("
    SELECT c.customer_id, c.full_name, lo.order_id, lo.order_date
    FROM customers c
    OUTER APPLY (
        SELECT o.order_id, o.order_date
        FROM orders o
        WHERE o.customer_id = c.customer_id
        ORDER BY o.order_date DESC
        FETCH FIRST 1 ROW ONLY
    ) lo
");
// Exactly 1 database round trip
```

### Common Laravel Mistakes with Oracle APPLY

- **Running a loop with one query per customer** — this is the N+1 pattern; use `CROSS APPLY` or `OUTER APPLY` instead
- **Loading all orders in PHP and filtering with `collect()->sortByDesc()->first()`** — this transfers massive amounts of data to PHP; let the database do it
- **Using `?` positional placeholders** — some Oracle PDO drivers work better with named parameters (`:param`); always test with your Oracle driver
- **Not indexing `customer_id` and `order_date` together** — a missing composite index makes `APPLY` orders of magnitude slower
- **Using `OUTER APPLY` when `CROSS APPLY` is sufficient** — if you only want rows that have children, `CROSS APPLY` is cleaner and signals intent
- **Confusing `OUTER APPLY` with `LEFT JOIN`** — they overlap but are not the same: `OUTER APPLY` handles correlated top-N; `LEFT JOIN` cannot

---

## 16. Practical Real-World Use Cases

`CROSS APPLY` and `OUTER APPLY` appear in many common business domains:

| Domain | Use Case | Construct |
|---|---|---|
| E-Commerce | Latest order per customer | `OUTER APPLY` |
| E-Commerce | Top-N products per order | `CROSS APPLY` |
| Finance | Latest account transaction per account | `OUTER APPLY` |
| Finance | Most recent balance per ledger entry | `OUTER APPLY` |
| HR | Latest salary record per employee | `OUTER APPLY` |
| HR | Most recent performance review per staff | `OUTER APPLY` |
| HR | Latest active contract per employee | `CROSS APPLY` |
| CRM | Latest address per customer | `OUTER APPLY` |
| CRM | Most recent contact activity per lead | `OUTER APPLY` |
| Helpdesk | Latest ticket status per customer | `OUTER APPLY` |
| Helpdesk | Oldest unresolved ticket per support agent | `CROSS APPLY` |
| Auth | Last login per user | `OUTER APPLY` |
| Auth | Last failed login attempt per user | `OUTER APPLY` |
| Audit | Most recent audit log entry per entity | `OUTER APPLY` |
| Procurement | Latest approval status per purchase request | `OUTER APPLY` |
| Procurement | Top-3 vendors by price per product | `CROSS APPLY` |
| Healthcare | Patient's most recent visit | `OUTER APPLY` |
| Healthcare | Latest lab result per patient per test type | `CROSS APPLY` |
| Education | Student's latest exam result per subject | `OUTER APPLY` |
| Education | Top-N students per class by score | `CROSS APPLY` |
| Logistics | Latest delivery status per shipment | `OUTER APPLY` |
| Inventory | Latest stock movement per product | `OUTER APPLY` |
| Insurance | Most recent claim per policy | `OUTER APPLY` |
| Subscriptions | Latest renewal per subscription | `OUTER APPLY` |

---

## 17. Comparison Tables

### 17.1 CROSS APPLY vs OUTER APPLY

| Feature | `CROSS APPLY` | `OUTER APPLY` |
|---|---|---|
| Left-side rows with no right match | **Excluded** | **Preserved (NULLs)** |
| Equivalent join type | INNER JOIN | LEFT OUTER JOIN |
| Use when | Right match is **mandatory** | Right match is **optional** |
| NULL in right columns | Never (row excluded) | Yes, when no match |
| Typical use | Active customers with orders | All customers including new ones |

### 17.2 CROSS APPLY vs INNER JOIN

| Feature | `CROSS APPLY` | `INNER JOIN` |
|---|---|---|
| ORDER BY inside | ✅ Supported | ❌ Not applicable |
| FETCH FIRST N inside | ✅ Supported | ❌ Not applicable |
| References outer columns | ✅ Lateral reference | ✅ (in ON condition) |
| Top-N per group | ✅ Native | ❌ Requires subquery + ROW_NUMBER |
| Simple flat join | Works | Cleaner, preferred |
| Performance (flat join) | Similar | Usually identical |

### 17.3 OUTER APPLY vs LEFT JOIN

| Feature | `OUTER APPLY` | `LEFT JOIN` |
|---|---|---|
| Preserves left rows with no match | ✅ | ✅ |
| ORDER BY inside | ✅ Supported | ❌ Not applicable |
| FETCH FIRST N inside | ✅ Supported | ❌ Not applicable |
| Returns N rows per left row | Controlled by FETCH FIRST | Returns ALL matching right rows |
| Duplicate left rows | Only if subquery returns multiple rows | Yes, one per matching right row |
| Top-1 or top-N per group | ✅ Native | ❌ Requires deduplication |

### 17.4 APPLY vs Correlated Subquery

| Feature | `APPLY` | Correlated Subquery |
|---|---|---|
| Returns multiple columns | ✅ | ❌ (one column only) |
| Returns multiple rows | ✅ | ❌ (one row only) |
| ORDER BY / FETCH FIRST | ✅ | ✅ (with limits) |
| Reuse inner result columns | ✅ | ❌ (repeat the subquery) |
| Table scans per outer row | 1 | 1 per column referenced |
| Readability | High | Low when many columns needed |

### 17.5 APPLY vs Window Functions

| Feature | `APPLY` | Window Functions (ROW_NUMBER) |
|---|---|---|
| Oracle version required | 12c R2+ | 8i+ |
| Top-N per group | ✅ Clean with FETCH FIRST | ✅ With ROW_NUMBER wrapping |
| Outer (null-preserved) behaviour | `OUTER APPLY` | LEFT JOIN + window in subquery |
| Aggregation across per-row logic | ✅ HAVING in subquery | Requires CTE or subquery |
| Performance: top-1 with index | Often better (stops early) | Can be slower (sorts all) |
| Performance: large N per group | Can be slower | Often better (parallel scan) |
| Readability | High | Moderate (requires wrapper) |

### 17.6 APPLY vs LATERAL

| Feature | `CROSS APPLY` / `OUTER APPLY` | `LATERAL` |
|---|---|---|
| Oracle support | 12c R2+ | 12c R1+ |
| SQL Server support | ✅ | ❌ |
| PostgreSQL support | ❌ | ✅ |
| Outer (null-preserved) form | `OUTER APPLY` | `LEFT JOIN LATERAL ... ON 1=1` |
| Readability | High (explicit inner/outer) | Moderate |
| Portability (Oracle + Postgres) | Oracle only | Prefer LATERAL |

### 17.7 Common Mistakes and Fixes

| Mistake | Fix |
|---|---|
| `COSS APPLY` (typo) | `CROSS APPLY` |
| Using `CROSS APPLY` but needing nulls for unmatched rows | Switch to `OUTER APPLY` |
| Using `OUTER APPLY` when you want to exclude non-matches | Switch to `CROSS APPLY` |
| Missing `ORDER BY` with `FETCH FIRST` | Always add `ORDER BY` before `FETCH FIRST`; result is non-deterministic without it |
| No index on correlated column | Add composite index: `(customer_id, order_date DESC)` |
| Referencing a right-side alias that does not exist in the outer SELECT | Use the correct `APPLY` alias (e.g., `lo.order_id`, not `o.order_id`) |
| Multiple APPLY blocks with duplicate column names | Alias all output columns explicitly |
| `APPLY` for a simple non-correlated join | Use a standard `JOIN` instead |
| Confusing Oracle syntax with SQL Server for `LATERAL` | Oracle uses `LATERAL`; SQL Server does not support `LATERAL` |
| Calling an expensive function inside the APPLY | Move the function outside, or cache its result using a CTE |

---

## 18. Common Mistakes

### Mistake 1: Typo — `COSS APPLY`

```sql
-- ❌ WRONG
SELECT * FROM customers c COSS APPLY (...) x;

-- ✅ CORRECT
SELECT * FROM customers c CROSS APPLY (...) x;
```

### Mistake 2: Using `CROSS APPLY` When All Rows Are Needed

```sql
-- ❌ WRONG: Eve and Frank disappear because they have no orders
SELECT c.full_name, lo.order_date
FROM customers c
CROSS APPLY (
    SELECT o.order_date FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC
    FETCH FIRST 1 ROW ONLY
) lo;

-- ✅ CORRECT: Use OUTER APPLY to preserve all customers
SELECT c.full_name, lo.order_date
FROM customers c
OUTER APPLY (
    SELECT o.order_date FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC
    FETCH FIRST 1 ROW ONLY
) lo;
```

### Mistake 3: No `ORDER BY` With `FETCH FIRST`

```sql
-- ❌ WRONG: which row is returned is unpredictable
SELECT c.full_name, o_result.order_id
FROM customers c
CROSS APPLY (
    SELECT o.order_id FROM orders o
    WHERE o.customer_id = c.customer_id
    FETCH FIRST 1 ROW ONLY    -- no ORDER BY: random result
) o_result;

-- ✅ CORRECT: always order before limiting
SELECT c.full_name, o_result.order_id
FROM customers c
CROSS APPLY (
    SELECT o.order_id FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC    -- deterministic: latest order
    FETCH FIRST 1 ROW ONLY
) o_result;
```

### Mistake 4: Forgetting That the Subquery Runs Per Row

```sql
-- Each customer causes a separate execution of the inner query.
-- Running APPLY on a 1-million-row outer table with no index
-- = 1 million full table scans of the inner table.
-- Always ensure the inner query is fast per individual execution.
```

### Mistake 5: Referencing the Wrong Alias

```sql
-- ❌ WRONG: 'o' is defined inside the subquery, not outside
SELECT c.full_name, o.order_date   -- 'o' is not in scope here
FROM customers c
CROSS APPLY (
    SELECT o.order_id, o.order_date
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC
    FETCH FIRST 1 ROW ONLY
) latest_order;

-- ✅ CORRECT: use the APPLY result alias
SELECT c.full_name, latest_order.order_date
FROM customers c
CROSS APPLY (
    SELECT o.order_id, o.order_date
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC
    FETCH FIRST 1 ROW ONLY
) latest_order;
```

### Mistake 6: Using APPLY for a Simple Join Without Row Limits

```sql
-- ❌ UNNECESSARY: no FETCH FIRST, no ORDER BY; use a regular JOIN
SELECT c.full_name, oa.order_id
FROM customers c
CROSS APPLY (
    SELECT o.order_id
    FROM orders o
    WHERE o.customer_id = c.customer_id
) oa;

-- ✅ BETTER: cleaner and the optimizer handles it better
SELECT c.full_name, o.order_id
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id;
```

### Mistake 7: Assuming APPLY Is Always Faster Than Window Functions

Both have their strengths. Always run `EXPLAIN PLAN` on your specific data volume. For large scans with many rows per group, `ROW_NUMBER()` with a proper covering index can outperform `APPLY`. For top-1 queries on well-indexed columns, `APPLY` typically wins.

---

## 19. Best Practices

- **Use `CROSS APPLY` when the right-side match is mandatory.** If rows without matching children should be excluded, `CROSS APPLY` communicates that intent clearly.
- **Use `OUTER APPLY` when all left-side rows must appear.** For dashboards, reports, and any query where zero matches should still produce a row with NULLs.
- **Always include a deterministic `ORDER BY` before `FETCH FIRST`.** Without it, the result is non-deterministic across executions. Use `ORDER BY primary_key DESC` as a tiebreaker.
- **Index the correlated column plus the ORDER BY column together.** The composite index `(customer_id, order_date DESC)` lets Oracle resolve the inner query in O(log n) without a sort step.
- **Keep the applied subquery selective.** Push all relevant filters inside the subquery, not outside in the outer `WHERE` clause. The inner query should discard as many rows as possible before the `FETCH FIRST` limit.
- **Use `APPLY` instead of multiple scalar subqueries.** When you need three or more values from the same correlated table, one `APPLY` block is faster and far cleaner than three separate correlated scalar subqueries.
- **Use `EXPLAIN PLAN` before deploying to production.** Verify the plan uses `INDEX RANGE SCAN` on correlated columns, not `TABLE ACCESS FULL`.
- **Compare against `ROW_NUMBER()`** for large datasets. For millions of rows, the window function approach may process data in parallel more effectively. Benchmark both.
- **Use bind parameters in application code.** Never concatenate user-supplied values into `APPLY` queries. Oracle's cursor cache works with bind parameters; hardcoded values cause cache thrashing.
- **Chain multiple `OUTER APPLY` blocks for dashboard queries.** Each block returns exactly one row (or NULL) per outer row, and later blocks can reference earlier blocks' results — a clean pattern for multi-facet dashboards.
- **Avoid `APPLY` on very small tables where a simple join is clearer.** `APPLY` adds cognitive overhead; only use it when its unique capabilities (correlated ORDER BY, FETCH FIRST, multi-column correlated results) are actually needed.

---

## 20. Conclusion

`CROSS APPLY` and `OUTER APPLY` are among the most powerful constructs in Oracle SQL for a specific but very common class of problems: **correlated, ordered, row-limited, or multi-column per-row queries**.

**Core summary:**

| Construct | Behaviour | Use When |
|---|---|---|
| `CROSS APPLY` | Inner join + correlated inline view | Right-side match is required |
| `OUTER APPLY` | Left join + correlated inline view | All left rows must be preserved |

They replace:
- Multiple correlated scalar subqueries (each hitting the same table)
- Complicated `ROW_NUMBER()` wrapping subqueries for top-N per group
- Inefficient PHP-side filtering of over-fetched result sets
- N+1 query patterns in application code

They are not always the right tool. A simple `INNER JOIN` or `LEFT JOIN` is better for flat, unrestricted relationships. A `ROW_NUMBER()` window function can outperform `APPLY` for large, unindexed datasets. Always use `EXPLAIN PLAN` to verify your chosen approach.

For **Laravel developers on Oracle**: embrace `DB::select()` with raw Oracle SQL for any query involving ordered correlated subqueries. The investment in writing clean Oracle SQL pays back immediately in reduced query counts, faster page loads, and simpler code compared to Eloquent loop workarounds.

The most important practical advice: index the correlated column and the ORDER BY column together as a composite index. This single decision is the difference between a sub-millisecond query and a query that times out in production.

---

*Oracle versions covered: 12c R2 (12.2.0.1) and later, including 18c, 19c, 21c, and 23ai. All SQL examples are written for Oracle syntax only and are not compatible with MySQL or standard ANSI SQL without modification.*
