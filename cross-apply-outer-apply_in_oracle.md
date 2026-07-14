# Oracle `CROSS APPLY` and `OUTER APPLY` — The Complete Advanced Guide

A deep, practical guide to Oracle's `CROSS APPLY`, `OUTER APPLY`, and `LATERAL` operators, with a full working e-commerce database, advanced real-world queries, performance tuning, and Laravel integration.

> **Target audience:** Developers (especially Laravel/PHP developers) working with Oracle Database 12c or later who want to master correlated row-by-row queries the right way.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [What Does `APPLY` Mean in SQL?](#2-what-does-apply-mean-in-sql)
3. [Why `APPLY` Is Useful in Oracle SQL](#3-why-apply-is-useful-in-oracle-sql)
4. [How the Right Side Can Reference the Left Side](#4-how-the-right-side-can-reference-the-left-side)
5. [Oracle Version Support](#5-oracle-version-support)
6. [Core Concept](#6-core-concept)
7. [Basic Syntax Explained Step by Step](#7-basic-syntax-explained-step-by-step)
8. [Full Working Database Setup (E-Commerce)](#8-full-working-database-setup-e-commerce)
9. [`CROSS APPLY` Examples](#9-cross-apply-examples)
10. [`OUTER APPLY` Examples](#10-outer-apply-examples)
11. [Comparison With Joins](#11-comparison-with-joins)
12. [Oracle `LATERAL` Explained](#12-oracle-lateral-explained)
13. [Advanced Business Scenario: Admin Dashboard](#13-advanced-business-scenario-admin-dashboard)
14. [Performance Tuning](#14-performance-tuning)
15. [Laravel Developer Section](#15-laravel-developer-section)
16. [Practical Real-Life Use Cases](#16-practical-real-life-use-cases)
17. [Comparison Tables](#17-comparison-tables)
18. [Common Mistakes](#18-common-mistakes)
19. [Best Practices](#19-best-practices)
20. [Conclusion](#20-conclusion)

---

## 1. Introduction

`CROSS APPLY` and `OUTER APPLY` are **lateral join operators**. They let you join a table to a **correlated subquery** — a subquery that can *see and use columns from the table on its left*.

A normal join joins two independent row sources. An `APPLY` runs the right-side query **once per row of the left-side table**, using that row's values as input. Conceptually:

```text
FOR EACH row in left_table:
    run right_query(using values from this left row)
    combine the left row with every row the right query returned
```

- **`CROSS APPLY`** keeps a left row **only if** the right-side query returned at least one row (like an `INNER JOIN`).
- **`OUTER APPLY`** keeps **every** left row; if the right-side query returned nothing, the right-side columns are `NULL` (like a `LEFT JOIN`).

This unlocks queries that are painful or impossible with plain joins, such as *"the latest order per customer"*, *"the top 3 products per order"*, or *"several aggregates per customer in one pass"* — all with clean, readable SQL.

Oracle added `CROSS APPLY` and `OUTER APPLY` (along with the ANSI-standard `LATERAL` keyword) in **Oracle Database 12c Release 1 (12.1)**, largely for compatibility with SQL Server, where `APPLY` has existed since 2005.

---

## 2. What Does `APPLY` Mean in SQL?

The word **apply** is used in the functional-programming sense: *apply this query (like a function) to each row of the left table*.

Think of the right-side query as a **parameterized table function**:

```text
latest_order(customer_id) -> returns 0 or 1 rows
top_products(order_id, n) -> returns up to n rows
```

`APPLY` calls that "function" for each left row and glues the results onto it:

| Operator | Behaves like | Left row kept when right returns nothing? |
|---|---|---|
| `CROSS APPLY` | `INNER JOIN` + correlated inline view | ❌ No — row is dropped |
| `OUTER APPLY` | `LEFT JOIN` + correlated inline view | ✅ Yes — right columns become `NULL` |

The key difference from a normal inline view: a normal inline view (`FROM (SELECT ...)`) **cannot** reference columns of other tables in the same `FROM` clause. An `APPLY`'d (lateral) inline view **can**.

```sql
-- ❌ ILLEGAL: normal inline view cannot see "c"
SELECT *
FROM customers c
JOIN (SELECT * FROM orders o WHERE o.customer_id = c.customer_id) x  -- ORA-00904
  ON 1 = 1;

-- ✅ LEGAL: CROSS APPLY makes the inline view "lateral" (correlated)
SELECT *
FROM customers c
CROSS APPLY (
    SELECT * FROM orders o WHERE o.customer_id = c.customer_id
) x;
```

---

## 3. Why `APPLY` Is Useful in Oracle SQL

`APPLY` shines whenever the right-side query needs **per-row logic** that a plain join condition cannot express. The most common needs:

- **`ORDER BY` inside the subquery** — e.g. sort a customer's orders by date to find the newest one.
- **`FETCH FIRST n ROWS ONLY`** — row-limited, per-parent results ("latest 1", "top 3").
- **Aggregation per left row** — compute `COUNT`, `SUM`, `AVG`, `MAX` for each parent in one correlated block.
- **Row-limited correlated results** — top-N-per-group without `ROW_NUMBER()` gymnastics.
- **Multiple calculated columns at once** — a correlated *scalar* subquery can return only **one** column; an `APPLY` block can return **many** columns from a single correlated execution.
- **Complex per-row logic** — filtering, `CASE` logic, joins, and transformations that depend on the current left row.

A single sentence to remember:

> **Use `APPLY` when you would love to write a correlated subquery, but you need it to return multiple columns and/or multiple ordered, row-limited rows.**

---

## 4. How the Right Side Can Reference the Left Side

Inside the applied subquery, you can reference **any column of any table that appears earlier (to the left) in the `FROM` clause**. This is called **lateral correlation**.

```sql
SELECT c.customer_name,
       lo.order_id,
       lo.order_date,
       lo.total_amount
FROM customers c
CROSS APPLY (
    SELECT o.order_id, o.order_date, o.total_amount
    FROM   orders o
    WHERE  o.customer_id = c.customer_id      -- 👈 references the LEFT table
    AND    o.status      <> 'CANCELLED'       -- normal filtering still works
    ORDER  BY o.order_date DESC
    FETCH  FIRST 1 ROW ONLY                   -- per-customer row limiting
) lo;
```

Rules that matter in practice:

1. The correlation reference (`c.customer_id`) may appear in `WHERE`, `SELECT`, `ON` clauses of joins *inside* the subquery, `CASE` expressions — almost anywhere.
2. Correlation flows **left to right only**. The left table cannot reference the applied subquery's columns inside its own definition, and an `APPLY` block cannot reference another `FROM` item that appears *after* it.
3. You can chain multiple `APPLY` blocks, and a later block can reference **both** the base table and earlier `APPLY` blocks:

```sql
FROM customers c
OUTER APPLY ( ...latest order for c...   ) lo
OUTER APPLY ( ...latest payment for lo... ) lp   -- lp can use both c.* and lo.*
```

This chaining ability is one of the biggest practical advantages of `APPLY` for dashboards and reports.

---

## 5. Oracle Version Support

| Feature | Introduced in Oracle | Notes |
|---|---|---|
| `CROSS APPLY` | **12c Release 1 (12.1)** | Oracle extension, matches SQL Server semantics |
| `OUTER APPLY` | **12c Release 1 (12.1)** | Oracle extension, matches SQL Server semantics |
| `LATERAL` (inline view) | **12c Release 1 (12.1)** documented/supported (existed earlier as undocumented behavior) | ANSI SQL standard keyword |
| `FETCH FIRST n ROWS ONLY` | **12c Release 1 (12.1)** | The row-limiting clause `APPLY` is usually combined with |

Practical implications:

- On **Oracle 11g or earlier**, none of these are available — you must use `ROW_NUMBER()` in a subquery, correlated scalar subqueries, or `ROWNUM` tricks.
- On **12.1+**, all three work. `LATERAL` is the ANSI-standard spelling and is the most portable (PostgreSQL, MySQL 8+, DB2 also support `LATERAL`); `CROSS APPLY`/`OUTER APPLY` is the spelling shared with SQL Server.
- Semantically in Oracle: `CROSS APPLY (subquery)` ≈ `CROSS JOIN LATERAL (subquery)` and `OUTER APPLY (subquery)` ≈ `LEFT OUTER JOIN LATERAL (subquery) ON 1=1`.

---

## 6. Core Concept

Burn these six statements into memory:

1. **`CROSS APPLY` works like an `INNER JOIN` with a correlated inline view.** The right-side subquery is evaluated per left row; only left rows with at least one right-side result survive.
2. **`OUTER APPLY` works like a `LEFT JOIN` with a correlated inline view.** Every left row survives; missing right-side results appear as `NULL`s.
3. **`CROSS APPLY` filters.** If your applied subquery finds nothing for a customer, that customer disappears from the output. This is sometimes exactly what you want ("only customers who ordered") and sometimes a nasty bug ("why did half my customers vanish from the dashboard?").
4. **`OUTER APPLY` preserves.** Perfect for dashboards, reports, and exports where every parent row must appear.
5. **The right side is a real query.** It can contain `ORDER BY`, `FETCH FIRST`, `GROUP BY`, joins, `CASE`, analytic functions — anything a standalone query can, *plus* references to left-side columns.
6. **`APPLY` returns multiple columns and multiple rows per parent.** That is what separates it from a correlated scalar subquery (1 column, 1 row) and makes it ideal for latest-row and top-N-per-group problems.

Mental model as a table:

| You need... | Right tool |
|---|---|
| 1 correlated value (e.g. order count) in `SELECT` | Correlated scalar subquery *or* `APPLY` |
| Several correlated values from the same lookup | ✅ `APPLY` (one execution, many columns) |
| Latest/first row per parent, parent must have one | ✅ `CROSS APPLY` + `ORDER BY` + `FETCH FIRST 1 ROW ONLY` |
| Latest/first row per parent, keep all parents | ✅ `OUTER APPLY` + `ORDER BY` + `FETCH FIRST 1 ROW ONLY` |
| Top-N rows per parent | ✅ `CROSS APPLY` / `OUTER APPLY` + `FETCH FIRST n ROWS ONLY` |
| Simple 1:1 / 1:N matching on keys, no ordering/limits | Plain `INNER JOIN` / `LEFT JOIN` |

---

## 7. Basic Syntax Explained Step by Step

### 7.1 `CROSS APPLY`

```sql
SELECT *
FROM parent_table p
CROSS APPLY (
    SELECT *
    FROM child_table c
    WHERE c.parent_id = p.parent_id
) child_result;
```

Step by step:

1. **`FROM parent_table p`** — the driving (left) row source. Alias `p` is what the subquery will reference.
2. **`CROSS APPLY ( ... )`** — declares a lateral, correlated inline view. Oracle logically evaluates the parenthesized query **once for each row of `p`**.
3. **`WHERE c.parent_id = p.parent_id`** — the correlation predicate. `p.parent_id` is the *current* parent row's value. This reference is legal **only because** of `APPLY`/`LATERAL`; in a plain inline view it would raise `ORA-00904: invalid identifier`.
4. **`child_result`** — the alias of the applied block. You use it in the outer `SELECT` like a table alias: `child_result.column_name`. (You may also write `child_result(col1, col2, ...)` to rename its columns.)
5. **Row logic** — if the subquery returns 3 rows for a given parent, that parent appears 3 times in the output. If it returns **0 rows, the parent is dropped**.

### 7.2 `OUTER APPLY`

```sql
SELECT *
FROM parent_table p
OUTER APPLY (
    SELECT *
    FROM child_table c
    WHERE c.parent_id = p.parent_id
) child_result;
```

Identical mechanics, with one change in step 5:

5. **Row logic** — if the subquery returns 0 rows for a given parent, the parent is **still returned once**, and every column of `child_result` is `NULL` for that row. This mirrors `LEFT JOIN` semantics.

### 7.3 The pattern you will use 90% of the time

```sql
SELECT p.*, x.*
FROM   parent_table p
OUTER APPLY (                    -- or CROSS APPLY to require a match
    SELECT c.col_a, c.col_b
    FROM   child_table c
    WHERE  c.parent_id = p.parent_id
    ORDER  BY c.created_at DESC  -- deterministic ordering
    FETCH  FIRST 1 ROW ONLY      -- "the latest one"
) x;
```

Note there is **no `ON` clause**. The "join condition" lives *inside* the subquery as an ordinary `WHERE` predicate.

---

## 8. Full Working Database Setup (E-Commerce)

A complete, runnable schema for an **e-commerce order management system**. All statements use Oracle 12c+ syntax (identity columns, `FETCH FIRST`, etc.).

### 8.1 Tables

```sql
-- Drop in dependency order if re-running (ignore errors on first run)
-- DROP TABLE audit_logs; DROP TABLE customer_support_tickets;
-- DROP TABLE payments;   DROP TABLE order_items;
-- DROP TABLE orders;     DROP TABLE products; DROP TABLE customers;

CREATE TABLE customers (
    customer_id    NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    customer_name  VARCHAR2(100) NOT NULL,
    email          VARCHAR2(150) NOT NULL UNIQUE,
    country        VARCHAR2(60)  DEFAULT 'Bangladesh' NOT NULL,
    risk_flag      CHAR(1)       DEFAULT 'N' NOT NULL,
    created_at     DATE          DEFAULT SYSDATE NOT NULL,
    CONSTRAINT chk_customers_risk CHECK (risk_flag IN ('Y', 'N'))
);

CREATE TABLE products (
    product_id     NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    product_name   VARCHAR2(120) NOT NULL,
    category       VARCHAR2(60)  NOT NULL,
    unit_price     NUMBER(12,2)  NOT NULL,
    active_flag    CHAR(1)       DEFAULT 'Y' NOT NULL,
    CONSTRAINT chk_products_price  CHECK (unit_price >= 0),
    CONSTRAINT chk_products_active CHECK (active_flag IN ('Y', 'N'))
);

CREATE TABLE orders (
    order_id       NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    customer_id    NUMBER        NOT NULL,
    order_date     DATE          DEFAULT SYSDATE NOT NULL,
    status         VARCHAR2(20)  DEFAULT 'PENDING' NOT NULL,
    total_amount   NUMBER(14,2)  NOT NULL,
    CONSTRAINT fk_orders_customer
        FOREIGN KEY (customer_id) REFERENCES customers (customer_id),
    CONSTRAINT chk_orders_status
        CHECK (status IN ('PENDING','CONFIRMED','SHIPPED','DELIVERED','CANCELLED')),
    CONSTRAINT chk_orders_amount CHECK (total_amount >= 0)
);

CREATE TABLE order_items (
    order_item_id  NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    order_id       NUMBER        NOT NULL,
    product_id     NUMBER        NOT NULL,
    quantity       NUMBER(6)     NOT NULL,
    unit_price     NUMBER(12,2)  NOT NULL,
    line_amount    NUMBER(14,2)  GENERATED ALWAYS AS (quantity * unit_price) VIRTUAL,
    CONSTRAINT fk_items_order   FOREIGN KEY (order_id)   REFERENCES orders (order_id),
    CONSTRAINT fk_items_product FOREIGN KEY (product_id) REFERENCES products (product_id),
    CONSTRAINT chk_items_qty    CHECK (quantity > 0),
    CONSTRAINT chk_items_price  CHECK (unit_price >= 0)
);

CREATE TABLE payments (
    payment_id     NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    order_id       NUMBER        NOT NULL,
    payment_date   DATE          DEFAULT SYSDATE NOT NULL,
    amount         NUMBER(14,2)  NOT NULL,
    method         VARCHAR2(20)  NOT NULL,
    status         VARCHAR2(20)  NOT NULL,
    CONSTRAINT fk_payments_order FOREIGN KEY (order_id) REFERENCES orders (order_id),
    CONSTRAINT chk_payments_method
        CHECK (method IN ('CARD','BKASH','NAGAD','BANK','COD')),
    CONSTRAINT chk_payments_status
        CHECK (status IN ('SUCCESS','FAILED','PENDING','REFUNDED')),
    CONSTRAINT chk_payments_amount CHECK (amount > 0)
);

CREATE TABLE customer_support_tickets (
    ticket_id      NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    customer_id    NUMBER        NOT NULL,
    subject        VARCHAR2(200) NOT NULL,
    status         VARCHAR2(20)  DEFAULT 'OPEN' NOT NULL,
    priority       VARCHAR2(10)  DEFAULT 'MEDIUM' NOT NULL,
    created_at     DATE          DEFAULT SYSDATE NOT NULL,
    CONSTRAINT fk_tickets_customer
        FOREIGN KEY (customer_id) REFERENCES customers (customer_id),
    CONSTRAINT chk_tickets_status
        CHECK (status IN ('OPEN','IN_PROGRESS','RESOLVED','CLOSED')),
    CONSTRAINT chk_tickets_priority CHECK (priority IN ('LOW','MEDIUM','HIGH'))
);

CREATE TABLE audit_logs (
    log_id         NUMBER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    entity_type    VARCHAR2(40)  NOT NULL,     -- 'ORDER', 'PAYMENT', 'TICKET', ...
    entity_id      NUMBER        NOT NULL,
    action         VARCHAR2(40)  NOT NULL,     -- 'CREATED', 'STATUS_CHANGED', ...
    performed_by   VARCHAR2(60)  NOT NULL,
    logged_at      TIMESTAMP     DEFAULT SYSTIMESTAMP NOT NULL
);
```

### 8.2 Sample Data

The data deliberately includes: customers **with and without** orders, orders **with and without** payments, and customers **with and without** support tickets — so `CROSS APPLY` vs `OUTER APPLY` behavior is visible.

```sql
-- Customers (5 & 6 have NO orders)
INSERT INTO customers (customer_id, customer_name, email, country, risk_flag, created_at) VALUES
    (1, 'Rahim Traders',   'rahim@example.com',  'Bangladesh', 'N', DATE '2025-01-05');
INSERT INTO customers (customer_id, customer_name, email, country, risk_flag, created_at) VALUES
    (2, 'Karima Begum',    'karima@example.com', 'Bangladesh', 'N', DATE '2025-01-12');
INSERT INTO customers (customer_id, customer_name, email, country, risk_flag, created_at) VALUES
    (3, 'Dhaka Gadgets',   'dg@example.com',     'Bangladesh', 'Y', DATE '2025-02-01');
INSERT INTO customers (customer_id, customer_name, email, country, risk_flag, created_at) VALUES
    (4, 'Sylhet Fashion',  'sf@example.com',     'Bangladesh', 'N', DATE '2025-02-20');
INSERT INTO customers (customer_id, customer_name, email, country, risk_flag, created_at) VALUES
    (5, 'New Signup Nila', 'nila@example.com',   'Bangladesh', 'N', DATE '2025-06-01');  -- no orders
INSERT INTO customers (customer_id, customer_name, email, country, risk_flag, created_at) VALUES
    (6, 'Ghost Account',   'ghost@example.com',  'Bangladesh', 'Y', DATE '2025-06-10');  -- no orders

-- Products (product 6 is NEVER sold)
INSERT INTO products (product_id, product_name, category, unit_price) VALUES (1, 'Wireless Mouse',      'Accessories', 1200);
INSERT INTO products (product_id, product_name, category, unit_price) VALUES (2, 'Mechanical Keyboard', 'Accessories', 4500);
INSERT INTO products (product_id, product_name, category, unit_price) VALUES (3, '27in Monitor',        'Displays',    28500);
INSERT INTO products (product_id, product_name, category, unit_price) VALUES (4, 'USB-C Hub',           'Accessories', 2300);
INSERT INTO products (product_id, product_name, category, unit_price) VALUES (5, 'Laptop Stand',        'Accessories', 1800);
INSERT INTO products (product_id, product_name, category, unit_price) VALUES (6, 'Legacy VGA Cable',    'Cables',      350);   -- never sold

-- Orders (customer 4's order 106 is CANCELLED; order 105 has NO payment)
INSERT INTO orders (order_id, customer_id, order_date, status, total_amount) VALUES
    (101, 1, DATE '2025-03-01', 'DELIVERED', 34200);
INSERT INTO orders (order_id, customer_id, order_date, status, total_amount) VALUES
    (102, 1, DATE '2025-05-15', 'SHIPPED',   6800);
INSERT INTO orders (order_id, customer_id, order_date, status, total_amount) VALUES
    (103, 2, DATE '2025-04-10', 'DELIVERED', 28500);
INSERT INTO orders (order_id, customer_id, order_date, status, total_amount) VALUES
    (104, 3, DATE '2025-05-01', 'CONFIRMED', 9100);
INSERT INTO orders (order_id, customer_id, order_date, status, total_amount) VALUES
    (105, 3, DATE '2025-06-20', 'PENDING',   4500);   -- no payment yet
INSERT INTO orders (order_id, customer_id, order_date, status, total_amount) VALUES
    (106, 4, DATE '2025-05-25', 'CANCELLED', 1200);   -- no payment

-- Order items
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES (101, 3, 1, 28500);
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES (101, 2, 1, 4500);
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES (101, 1, 1, 1200);
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES (102, 4, 2, 2300);
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES (102, 1, 1, 1200);
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES (102, 5, 1, 1000);  -- discounted stand
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES (103, 3, 1, 28500);
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES (104, 2, 2, 4550);
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES (105, 2, 1, 4500);
INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES (106, 1, 1, 1200);

-- Payments (order 101 has a FAILED then a SUCCESS payment; 105 & 106 unpaid)
INSERT INTO payments (order_id, payment_date, amount, method, status) VALUES
    (101, DATE '2025-03-01', 34200, 'CARD',  'FAILED');
INSERT INTO payments (order_id, payment_date, amount, method, status) VALUES
    (101, DATE '2025-03-02', 34200, 'BKASH', 'SUCCESS');
INSERT INTO payments (order_id, payment_date, amount, method, status) VALUES
    (102, DATE '2025-05-15', 6800,  'NAGAD', 'SUCCESS');
INSERT INTO payments (order_id, payment_date, amount, method, status) VALUES
    (103, DATE '2025-04-10', 28500, 'CARD',  'SUCCESS');
INSERT INTO payments (order_id, payment_date, amount, method, status) VALUES
    (104, DATE '2025-05-02', 9100,  'BANK',  'PENDING');

-- Support tickets (customers 2 and 3 have tickets; 1, 4, 5, 6 do not)
INSERT INTO customer_support_tickets (customer_id, subject, status, priority, created_at) VALUES
    (2, 'Monitor arrived with dead pixels', 'RESOLVED',    'HIGH',   DATE '2025-04-15');
INSERT INTO customer_support_tickets (customer_id, subject, status, priority, created_at) VALUES
    (3, 'Payment stuck in pending',         'IN_PROGRESS', 'HIGH',   DATE '2025-05-05');
INSERT INTO customer_support_tickets (customer_id, subject, status, priority, created_at) VALUES
    (3, 'Change delivery address',          'OPEN',        'MEDIUM', DATE '2025-06-21');

-- Audit logs
INSERT INTO audit_logs (entity_type, entity_id, action, performed_by) VALUES ('ORDER',   101, 'CREATED',        'system');
INSERT INTO audit_logs (entity_type, entity_id, action, performed_by) VALUES ('ORDER',   101, 'STATUS_CHANGED', 'admin_rifat');
INSERT INTO audit_logs (entity_type, entity_id, action, performed_by) VALUES ('PAYMENT', 2,   'CREATED',        'gateway');
INSERT INTO audit_logs (entity_type, entity_id, action, performed_by) VALUES ('TICKET',  2,   'CREATED',        'support_bot');

COMMIT;
```

### 8.3 Data Landscape at a Glance

| Customer | Orders | Payments on those orders | Tickets |
|---|---|---|---|
| 1 — Rahim Traders | 101, 102 | 101 ✅ (after a failure), 102 ✅ | none |
| 2 — Karima Begum | 103 | ✅ | 1 resolved |
| 3 — Dhaka Gadgets | 104, 105 | 104 pending, **105 none** | 2 (open + in progress) |
| 4 — Sylhet Fashion | 106 (cancelled) | **none** | none |
| 5 — New Signup Nila | **none** | — | none |
| 6 — Ghost Account | **none** | — | none |

---

## 9. `CROSS APPLY` Examples

> Reminder: `CROSS APPLY` **drops** left rows with no match. Every example below intentionally excludes some rows.

### 9.1 Latest Order Per Customer (only customers who ordered)

```sql
SELECT c.customer_id,
       c.customer_name,
       lo.order_id,
       lo.order_date,
       lo.status,
       lo.total_amount
FROM   customers c
CROSS APPLY (
    SELECT o.order_id, o.order_date, o.status, o.total_amount
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
    ORDER  BY o.order_date DESC, o.order_id DESC   -- tie-breaker => deterministic
    FETCH  FIRST 1 ROW ONLY
) lo
ORDER BY c.customer_id;
```

Expected output — note customers 5 and 6 are **gone**:

| CUSTOMER_ID | CUSTOMER_NAME | ORDER_ID | ORDER_DATE | STATUS | TOTAL_AMOUNT |
|---|---|---|---|---|---|
| 1 | Rahim Traders | 102 | 2025-05-15 | SHIPPED | 6800 |
| 2 | Karima Begum | 103 | 2025-04-10 | DELIVERED | 28500 |
| 3 | Dhaka Gadgets | 105 | 2025-06-20 | PENDING | 4500 |
| 4 | Sylhet Fashion | 106 | 2025-05-25 | CANCELLED | 1200 |

Step by step:

1. For each customer row, Oracle runs the inner query with that customer's `customer_id`.
2. `ORDER BY o.order_date DESC, o.order_id DESC` sorts that customer's orders newest-first; the `order_id` tie-breaker guarantees a deterministic winner if two orders share a date.
3. `FETCH FIRST 1 ROW ONLY` keeps only the newest.
4. Customers with zero orders produce zero right-side rows → `CROSS APPLY` removes them.

### 9.2 Top 3 Products Per Order (by line amount)

```sql
SELECT o.order_id,
       o.order_date,
       t.product_name,
       t.quantity,
       t.line_amount,
       t.line_rank
FROM   orders o
CROSS APPLY (
    SELECT p.product_name,
           oi.quantity,
           oi.line_amount,
           ROW_NUMBER() OVER (ORDER BY oi.line_amount DESC) AS line_rank
    FROM   order_items oi
    JOIN   products p ON p.product_id = oi.product_id     -- join INSIDE the applied block
    WHERE  oi.order_id = o.order_id                       -- correlation
    ORDER  BY oi.line_amount DESC
    FETCH  FIRST 3 ROWS ONLY
) t
ORDER BY o.order_id, t.line_rank;
```

Key points:

- The applied block contains its **own join** (`order_items` → `products`) — perfectly legal.
- `FETCH FIRST 3 ROWS ONLY` limits per-order, not globally. A plain join cannot do this.
- Order 102 has 3 items → 3 rows; order 103 has 1 item → 1 row. Per-parent limits adapt automatically.

### 9.3 Latest Successful Payment Per Order (only paid orders)

```sql
SELECT o.order_id,
       o.status          AS order_status,
       o.total_amount,
       sp.payment_id,
       sp.payment_date,
       sp.method,
       sp.amount         AS paid_amount
FROM   orders o
CROSS APPLY (
    SELECT p.payment_id, p.payment_date, p.method, p.amount
    FROM   payments p
    WHERE  p.order_id = o.order_id
    AND    p.status   = 'SUCCESS'          -- extra correlated filter
    ORDER  BY p.payment_date DESC, p.payment_id DESC
    FETCH  FIRST 1 ROW ONLY
) sp
ORDER BY o.order_id;
```

Result: only orders 101, 102, 103 appear. Order 101's **FAILED** attempt is filtered out inside the block, so its SUCCESS payment (BKASH) is chosen. Orders 104 (pending), 105 and 106 (no payments) are dropped — exactly the point of `CROSS APPLY` here: *"show me only orders that actually collected money."*

### 9.4 Customer's Most Recent Support Ticket (only customers with tickets)

```sql
SELECT c.customer_id,
       c.customer_name,
       lt.ticket_id,
       lt.subject,
       lt.status,
       lt.priority,
       lt.created_at
FROM   customers c
CROSS APPLY (
    SELECT t.ticket_id, t.subject, t.status, t.priority, t.created_at
    FROM   customer_support_tickets t
    WHERE  t.customer_id = c.customer_id
    ORDER  BY t.created_at DESC, t.ticket_id DESC
    FETCH  FIRST 1 ROW ONLY
) lt
ORDER BY c.customer_id;
```

Only customers 2 and 3 appear. For customer 3 (two tickets) the newer *"Change delivery address"* ticket wins.

### 9.5 Correlated Aggregation Per Customer

One applied block returning **four aggregates at once** — something a correlated scalar subquery would need four separate subqueries for:

```sql
SELECT c.customer_id,
       c.customer_name,
       m.total_orders,
       m.total_purchase_amount,
       m.last_order_date,
       m.avg_order_amount
FROM   customers c
CROSS APPLY (
    SELECT COUNT(*)                    AS total_orders,
           SUM(o.total_amount)         AS total_purchase_amount,
           MAX(o.order_date)           AS last_order_date,
           ROUND(AVG(o.total_amount),2) AS avg_order_amount
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
    AND    o.status <> 'CANCELLED'      -- business rule: ignore cancelled orders
    HAVING COUNT(*) > 0                 -- 👈 without this, CROSS APPLY would keep everyone
) m
ORDER BY m.total_purchase_amount DESC;
```

⚠️ **Subtle trap explained:** an aggregate query with no `GROUP BY` **always returns exactly one row**, even when it matches nothing (`COUNT(*) = 0`, sums are `NULL`). So without the `HAVING COUNT(*) > 0`, `CROSS APPLY` would behave like `OUTER APPLY` here and keep order-less customers with zeroed metrics. The `HAVING` clause restores true "must have orders" semantics. If you *want* every customer with zeros, drop the `HAVING` (or use `OUTER APPLY` with `NVL`).

---

## 10. `OUTER APPLY` Examples

> Reminder: `OUTER APPLY` **preserves** every left row. Missing right-side data appears as `NULL`.

### 10.1 All Customers With Latest Order If Available

```sql
SELECT c.customer_id,
       c.customer_name,
       lo.order_id,
       lo.order_date,
       lo.total_amount,
       CASE WHEN lo.order_id IS NULL THEN 'NEVER ORDERED' ELSE 'HAS ORDERS' END AS order_state
FROM   customers c
OUTER APPLY (
    SELECT o.order_id, o.order_date, o.total_amount
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
    ORDER  BY o.order_date DESC, o.order_id DESC
    FETCH  FIRST 1 ROW ONLY
) lo
ORDER BY c.customer_id;
```

Expected output — customers 5 and 6 now **appear with NULLs**:

| CUSTOMER_ID | CUSTOMER_NAME | ORDER_ID | ORDER_DATE | TOTAL_AMOUNT | ORDER_STATE |
|---|---|---|---|---|---|
| 1 | Rahim Traders | 102 | 2025-05-15 | 6800 | HAS ORDERS |
| 2 | Karima Begum | 103 | 2025-04-10 | 28500 | HAS ORDERS |
| 3 | Dhaka Gadgets | 105 | 2025-06-20 | 4500 | HAS ORDERS |
| 4 | Sylhet Fashion | 106 | 2025-05-25 | 1200 | HAS ORDERS |
| 5 | New Signup Nila | *(null)* | *(null)* | *(null)* | NEVER ORDERED |
| 6 | Ghost Account | *(null)* | *(null)* | *(null)* | NEVER ORDERED |

Compare with example 9.1: the query body is identical; only `CROSS` → `OUTER` changed, and the result set grew from 4 to 6 rows.

### 10.2 All Orders With Latest Payment If Available

```sql
SELECT o.order_id,
       o.order_date,
       o.status              AS order_status,
       o.total_amount,
       lp.payment_id,
       lp.payment_date,
       lp.method,
       lp.status             AS payment_status,
       NVL(lp.status, 'NO PAYMENT') AS payment_state
FROM   orders o
OUTER APPLY (
    SELECT p.payment_id, p.payment_date, p.method, p.status
    FROM   payments p
    WHERE  p.order_id = o.order_id
    ORDER  BY p.payment_date DESC, p.payment_id DESC
    FETCH  FIRST 1 ROW ONLY
) lp
ORDER BY o.order_id;
```

Orders 105 and 106 show `NO PAYMENT`; order 101 shows its **latest** payment (the SUCCESS one from 2025-03-02, because we ordered by date descending — not because we filtered on status).

### 10.3 All Customers With Last Support Ticket If Available

```sql
SELECT c.customer_id,
       c.customer_name,
       lt.ticket_id,
       lt.subject,
       NVL(lt.status, 'NO TICKETS') AS ticket_status,
       lt.created_at
FROM   customers c
OUTER APPLY (
    SELECT t.ticket_id, t.subject, t.status, t.created_at
    FROM   customer_support_tickets t
    WHERE  t.customer_id = c.customer_id
    ORDER  BY t.created_at DESC, t.ticket_id DESC
    FETCH  FIRST 1 ROW ONLY
) lt
ORDER BY c.customer_id;
```

All six customers appear; only 2 and 3 carry ticket data.

### 10.4 Customer Dashboard Query (chained `OUTER APPLY`)

One query, four correlated lookups, every customer preserved. Note how the third block (`lp`) references the **previous** applied block (`lo`) — chaining in action:

```sql
SELECT c.customer_id,
       c.customer_name,
       lo.order_date                     AS latest_order_date,
       lo.total_amount                   AS latest_order_amount,
       NVL(lp.status, 'NO PAYMENT')      AS last_payment_status,
       NVL(lt.status, 'NO TICKETS')      AS last_ticket_status,
       NVL(agg.total_order_value, 0)     AS total_order_value,
       NVL(agg.order_count, 0)           AS number_of_orders
FROM   customers c
OUTER APPLY (                                   -- latest order
    SELECT o.order_id, o.order_date, o.total_amount
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
    ORDER  BY o.order_date DESC, o.order_id DESC
    FETCH  FIRST 1 ROW ONLY
) lo
OUTER APPLY (                                   -- latest payment OF that latest order
    SELECT p.status
    FROM   payments p
    WHERE  p.order_id = lo.order_id             -- 👈 references previous APPLY block
    ORDER  BY p.payment_date DESC, p.payment_id DESC
    FETCH  FIRST 1 ROW ONLY
) lp
OUTER APPLY (                                   -- latest support ticket
    SELECT t.status
    FROM   customer_support_tickets t
    WHERE  t.customer_id = c.customer_id
    ORDER  BY t.created_at DESC, t.ticket_id DESC
    FETCH  FIRST 1 ROW ONLY
) lt
OUTER APPLY (                                   -- lifetime aggregates
    SELECT SUM(o.total_amount) AS total_order_value,
           COUNT(*)            AS order_count
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
    AND    o.status <> 'CANCELLED'
) agg
ORDER BY c.customer_id;
```

Why this beats the join alternative:

- With `LEFT JOIN`s you would join *all* orders, *all* payments, and *all* tickets, producing a **fan-out** (row multiplication) that corrupts `SUM`/`COUNT`, then need `DISTINCT` or pre-aggregated subqueries to fix it.
- Here each block independently returns **at most one row per customer**, so there is no fan-out and the aggregates stay correct.

### 10.5 Product Sales Summary (including never-sold products)

```sql
SELECT p.product_id,
       p.product_name,
       p.category,
       NVL(s.units_sold, 0)      AS units_sold,
       NVL(s.revenue, 0)         AS revenue,
       s.last_sold_date,
       CASE WHEN s.units_sold IS NULL THEN 'NEVER SOLD' ELSE 'SELLING' END AS sales_state
FROM   products p
OUTER APPLY (
    SELECT SUM(oi.quantity)    AS units_sold,
           SUM(oi.line_amount) AS revenue,
           MAX(o.order_date)   AS last_sold_date
    FROM   order_items oi
    JOIN   orders o ON o.order_id = oi.order_id
    WHERE  oi.product_id = p.product_id
    AND    o.status <> 'CANCELLED'
    HAVING COUNT(*) > 0        -- make "never sold" produce NULLs instead of a zero row
) s
ORDER BY revenue DESC;
```

Product 6 (*Legacy VGA Cable*) appears with `NEVER SOLD` — the exact rows an `INNER JOIN` report silently hides and a merchandising team desperately needs to see.

---

## 11. Comparison With Joins

### 11.1 `CROSS APPLY` vs `INNER JOIN`

**When they produce identical results:** when the applied subquery is a plain correlated filter with no `ORDER BY`, no `FETCH FIRST`, and no aggregation:

```sql
-- These two are equivalent:
SELECT c.customer_name, o.order_id
FROM   customers c
JOIN   orders o ON o.customer_id = c.customer_id;

SELECT c.customer_name, x.order_id
FROM   customers c
CROSS APPLY (
    SELECT o.order_id FROM orders o WHERE o.customer_id = c.customer_id
) x;
```

In this case, **use the `INNER JOIN`** — it is clearer and gives the optimizer maximum freedom (hash join, merge join, nested loops).

**When `CROSS APPLY` is better:** the moment per-parent `ORDER BY` + row limiting or per-parent aggregation enters the picture:

```sql
-- "Each customer's single latest non-cancelled order" — try this with a plain INNER JOIN.
SELECT c.customer_name, x.order_id, x.order_date
FROM   customers c
CROSS APPLY (
    SELECT o.order_id, o.order_date
    FROM   orders o
    WHERE  o.customer_id = c.customer_id AND o.status <> 'CANCELLED'
    ORDER  BY o.order_date DESC, o.order_id DESC
    FETCH  FIRST 1 ROW ONLY
) x;
```

A plain join has no concept of "1 row per parent, chosen by ordering" — you would need `ROW_NUMBER()` in a derived table instead.

### 11.2 `OUTER APPLY` vs `LEFT JOIN`

**Identical results** for plain correlated filters:

```sql
SELECT c.customer_name, o.order_id
FROM   customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id;
-- ≡
SELECT c.customer_name, x.order_id
FROM   customers c
OUTER APPLY (SELECT o.order_id FROM orders o WHERE o.customer_id = c.customer_id) x;
```

**`OUTER APPLY` wins** when you want *all parents plus their single latest/top child*:

```sql
-- LEFT JOIN version needs a pre-ranked derived table:
SELECT c.customer_name, o.order_id, o.order_date
FROM   customers c
LEFT JOIN (
    SELECT o.*, ROW_NUMBER() OVER (PARTITION BY o.customer_id
                                   ORDER BY o.order_date DESC, o.order_id DESC) rn
    FROM   orders o
) o ON o.customer_id = c.customer_id AND o.rn = 1;

-- OUTER APPLY version says what it means:
SELECT c.customer_name, x.order_id, x.order_date
FROM   customers c
OUTER APPLY (
    SELECT o.order_id, o.order_date
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
    ORDER  BY o.order_date DESC, o.order_id DESC
    FETCH  FIRST 1 ROW ONLY
) x;
```

Both are valid; the `APPLY` version is more direct, and when only a small subset of customers is queried (e.g. one dashboard page), it can be much faster because it never ranks the *entire* orders table.

### 11.3 `APPLY` vs Correlated Subquery

A correlated **scalar** subquery in the `SELECT` list returns exactly **one column** and at most **one row**. Needing four values means four subqueries — four separate correlated probes:

```sql
-- Painful: 4 correlated probes per customer
SELECT c.customer_name,
       (SELECT COUNT(*)          FROM orders o WHERE o.customer_id = c.customer_id) AS total_orders,
       (SELECT SUM(total_amount) FROM orders o WHERE o.customer_id = c.customer_id) AS total_amount,
       (SELECT MAX(order_date)   FROM orders o WHERE o.customer_id = c.customer_id) AS last_order,
       (SELECT AVG(total_amount) FROM orders o WHERE o.customer_id = c.customer_id) AS avg_amount
FROM   customers c;

-- Clean: 1 correlated block, 4 columns, reusable alias
SELECT c.customer_name, m.total_orders, m.total_amount, m.last_order, m.avg_amount
FROM   customers c
OUTER APPLY (
    SELECT COUNT(*)          AS total_orders,
           SUM(total_amount) AS total_amount,
           MAX(order_date)   AS last_order,
           AVG(total_amount) AS avg_amount
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
) m;
```

Beyond readability: the `APPLY` columns are **reusable** in the outer query (in `WHERE`, `ORDER BY`, expressions like `m.total_amount / NULLIF(m.total_orders,0)`) without repeating the subquery.

### 11.4 `APPLY` vs Window Functions (`ROW_NUMBER()`)

Both solve "latest row / top-N rows per group". Same problem, two idioms:

```sql
-- Using OUTER APPLY
SELECT c.customer_id, c.customer_name, x.order_id, x.order_date
FROM   customers c
OUTER APPLY (
    SELECT o.order_id, o.order_date
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
    ORDER  BY o.order_date DESC, o.order_id DESC
    FETCH  FIRST 1 ROW ONLY
) x;
```

```sql
-- Using ROW_NUMBER() OVER (PARTITION BY ...)
SELECT c.customer_id, c.customer_name, r.order_id, r.order_date
FROM   customers c
LEFT JOIN (
    SELECT o.customer_id, o.order_id, o.order_date,
           ROW_NUMBER() OVER (PARTITION BY o.customer_id
                              ORDER BY o.order_date DESC, o.order_id DESC) AS rn
    FROM   orders o
) r ON r.customer_id = c.customer_id AND r.rn = 1;
```

**Which is better when?**

| Situation | Prefer |
|---|---|
| Small/filtered set of parents (one page, one customer, a search result) | **`APPLY`** — indexed nested-loop probes touch only the needed child rows |
| All (or most) parents, huge child table, full report | **`ROW_NUMBER()`** — one full scan + window sort usually beats millions of index probes |
| Need columns from parent inside the child logic (dynamic filters, `CASE` on parent values) | **`APPLY`** — windows can't see the parent |
| Need rank numbers themselves in output | **`ROW_NUMBER()`** (or put it *inside* the `APPLY` block as in §9.2) |
| Pre-12c Oracle | **`ROW_NUMBER()`** — `APPLY` doesn't exist |
| Readability for "latest X per Y" | Subjective; `APPLY` reads top-down, windows require understanding partitioned ranking |

Rule of thumb: **`APPLY` for selective lookups, window functions for whole-table crunching** — and when in doubt, benchmark both with `EXPLAIN PLAN`.

---

## 12. Oracle `LATERAL` Explained

### 12.1 What `LATERAL` Means

`LATERAL` is the **ANSI SQL standard** keyword that marks an inline view as *laterally correlated* — allowed to reference columns from preceding items in the same `FROM` clause. It is the standardized concept that `CROSS APPLY`/`OUTER APPLY` implement with vendor-specific spelling.

### 12.2 How `LATERAL` Relates to `CROSS APPLY` / `OUTER APPLY`

In Oracle 12c+:

| Oracle/SQL Server spelling | ANSI spelling |
|---|---|
| `CROSS APPLY (subquery) x` | `CROSS JOIN LATERAL (subquery) x` — or comma-join `, LATERAL (subquery) x` |
| `OUTER APPLY (subquery) x` | `LEFT OUTER JOIN LATERAL (subquery) x ON 1 = 1` |

All three spellings compile to the same lateral-view mechanics in Oracle's optimizer.

### 12.3 The Same Query Three Ways

```sql
-- 1) CROSS APPLY
SELECT c.customer_name, x.order_id, x.order_date
FROM   customers c
CROSS APPLY (
    SELECT o.order_id, o.order_date
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
    ORDER  BY o.order_date DESC
    FETCH  FIRST 1 ROW ONLY
) x;
```

```sql
-- 2) OUTER APPLY (keeps customers with no orders)
SELECT c.customer_name, x.order_id, x.order_date
FROM   customers c
OUTER APPLY (
    SELECT o.order_id, o.order_date
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
    ORDER  BY o.order_date DESC
    FETCH  FIRST 1 ROW ONLY
) x;
```

```sql
-- 3) LATERAL (comma join = inner semantics, like CROSS APPLY)
SELECT c.customer_name, x.order_id, x.order_date
FROM   customers c,
LATERAL (
    SELECT o.order_id, o.order_date
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
    ORDER  BY o.order_date DESC
    FETCH  FIRST 1 ROW ONLY
) x;

-- 3b) LATERAL with outer semantics, like OUTER APPLY
SELECT c.customer_name, x.order_id, x.order_date
FROM   customers c
LEFT OUTER JOIN LATERAL (
    SELECT o.order_id, o.order_date
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
    ORDER  BY o.order_date DESC
    FETCH  FIRST 1 ROW ONLY
) x ON 1 = 1;
```

### 12.4 Readability and Portability

- **Portability:** `LATERAL` is standard SQL and works in PostgreSQL, MySQL 8+, DB2, and Oracle. `CROSS APPLY`/`OUTER APPLY` works in SQL Server, Oracle 12c+, and a few others. If your codebase may move between Oracle and PostgreSQL, prefer `LATERAL`; between Oracle and SQL Server, prefer `APPLY`.
- **Readability:** many developers find `OUTER APPLY` clearer than `LEFT OUTER JOIN LATERAL ... ON 1 = 1` — the dummy `ON 1 = 1` is noise required by ANSI join grammar. For inner semantics the two are about equally readable.
- **Team convention beats personal taste:** pick one spelling per codebase and stick to it.

---

## 13. Advanced Business Scenario: Admin Dashboard

**Scenario:** the e-commerce company's admin dashboard must show, for *every* customer: latest order, latest payment, latest support ticket, total order amount, and a computed risk status. Some customers have never ordered; some orders are unpaid. **No customer may be missing from the dashboard.**

Why plain joins fail here:

1. `INNER JOIN` to orders silently deletes customers 5 and 6 — new signups the sales team most wants to contact.
2. `LEFT JOIN` to *all* orders, *all* payments, *all* tickets multiplies rows (fan-out): customer 3 (2 orders × 2 tickets) becomes 4 rows, and any `SUM(total_amount)` double-counts.
3. Fixing the fan-out requires nested pre-aggregated/pre-ranked derived tables — the query becomes a wall of subqueries.

The `OUTER APPLY` solution stays flat, correct, and readable:

```sql
SELECT c.customer_id,
       c.customer_name,
       lo.order_date                       AS latest_order_date,
       lo.total_amount                     AS latest_order_amount,
       NVL(lp.status, 'NO PAYMENT')        AS latest_payment_status,
       NVL(lt.status, 'NO TICKETS')        AS latest_ticket_status,
       NVL(agg.total_order_value, 0)       AS total_order_value,
       CASE
           WHEN c.risk_flag = 'Y'                                   THEN 'FLAGGED'
           WHEN lp.status = 'FAILED'                                THEN 'PAYMENT RISK'
           WHEN lt.status IN ('OPEN','IN_PROGRESS')
                AND lt.priority = 'HIGH'                            THEN 'SUPPORT RISK'
           WHEN lo.order_id IS NULL                                 THEN 'NEW / INACTIVE'
           ELSE 'HEALTHY'
       END                                 AS risk_status
FROM   customers c
OUTER APPLY (
    SELECT o.order_id, o.order_date, o.total_amount
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
    ORDER  BY o.order_date DESC, o.order_id DESC
    FETCH  FIRST 1 ROW ONLY
) lo
OUTER APPLY (
    SELECT p.status
    FROM   payments p
    WHERE  p.order_id = lo.order_id
    ORDER  BY p.payment_date DESC, p.payment_id DESC
    FETCH  FIRST 1 ROW ONLY
) lp
OUTER APPLY (
    SELECT t.status, t.priority
    FROM   customer_support_tickets t
    WHERE  t.customer_id = c.customer_id
    ORDER  BY t.created_at DESC, t.ticket_id DESC
    FETCH  FIRST 1 ROW ONLY
) lt
OUTER APPLY (
    SELECT SUM(o.total_amount) AS total_order_value
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
    AND    o.status <> 'CANCELLED'
) agg
ORDER BY total_order_value DESC, c.customer_id;
```

What this demonstrates:

- **Preservation:** all 6 customers appear, including the two with no orders (`NEW / INACTIVE`).
- **No fan-out:** each block returns ≤ 1 row per customer, so one customer = one dashboard row, and totals are correct.
- **Chained correlation:** `lp` depends on `lo.order_id` — "the latest payment *of the latest order*", a relationship that is genuinely awkward to express with joins.
- **Business logic on top:** the `CASE` risk classifier freely mixes columns from the base table and all four applied blocks.

---

## 14. Performance Tuning

### 14.1 How Oracle Executes Correlated `APPLY` Queries

Conceptually, an `APPLY` is evaluated per left row. Physically, Oracle's optimizer has choices:

- **Nested loops (the common case):** for each left row, probe the right side. Cheap **if and only if** the probe is an index lookup returning few rows. Cost ≈ *(left rows) × (cost of one probe)*.
- **Lateral view decorrelation:** for some applied subqueries (especially aggregations without `FETCH FIRST`), the optimizer can transform the lateral view into an ordinary join or `GROUP BY` and use hash joins instead. `ORDER BY ... FETCH FIRST` inside the block usually prevents this, keeping nested loops.
- Check the plan for operations like `NESTED LOOPS`, `VIEW`, `VW_LAT_...` (internal lateral-view names), and `WINDOW SORT`/`SORT ORDER BY STOPKEY` under the lateral view.

The takeaway: **the applied subquery runs (logically) once per outer row, so its single-execution cost must be tiny.** A 2ms probe is invisible for 50 dashboard rows and catastrophic for 5 million.

### 14.2 Why Indexing Is Critical

Without an index on the correlated column, each probe is a **full table scan of the child table per parent row** — the classic way `APPLY` queries melt down. With a composite index matching *correlation column(s) + ORDER BY column(s)*, each probe becomes an index range scan that can stop after the first row.

### 14.3 Recommended Indexes for This Schema

```sql
CREATE INDEX ix_orders_cust_date   ON orders   (customer_id, order_date);
CREATE INDEX ix_payments_ord_date  ON payments (order_id, payment_date);
CREATE INDEX ix_items_ord_amount   ON order_items (order_id, line_amount);
CREATE INDEX ix_tickets_cust_date  ON customer_support_tickets (customer_id, created_at);
```

Design logic (using `ix_orders_cust_date` as the model):

1. **Leading column = correlation column** (`customer_id`) — lets Oracle jump straight to that customer's slice of the index.
2. **Second column = the `ORDER BY` column** (`order_date`) — that slice is already sorted, so *no sort is needed*.
3. Optionally use `DESC` (`order_date DESC`) or add frequently selected columns to make the index covering and avoid table access entirely.

### 14.4 How `FETCH FIRST 1 ROW ONLY` Is Optimized

With the composite index above, the plan for the "latest order" block becomes:

```text
INDEX RANGE SCAN DESCENDING on ix_orders_cust_date  (stops after 1 row — COUNT STOPKEY /
TABLE ACCESS BY INDEX ROWID (1 row)                  SORT ORDER BY STOPKEY in the plan)
```

Oracle descends the index to `(customer_id, MAX(order_date))`, reads **one index entry**, fetches **one table row**, and stops. Latest-row lookup in a handful of logical I/Os, regardless of how many orders the customer has. Without the index: full scan + sort of all the customer's orders, per customer.

### 14.5 Inspecting Execution Plans

```sql
EXPLAIN PLAN FOR
SELECT c.customer_name, x.order_id, x.order_date
FROM   customers c
OUTER APPLY (
    SELECT o.order_id, o.order_date
    FROM   orders o
    WHERE  o.customer_id = c.customer_id
    ORDER  BY o.order_date DESC, o.order_id DESC
    FETCH  FIRST 1 ROW ONLY
) x;
```

```sql
SELECT *
FROM TABLE(DBMS_XPLAN.DISPLAY);
```

What to look for:

- `NESTED LOOPS OUTER` driving into a `VIEW VW_LAT_xxx` — the lateral view.
- `INDEX RANGE SCAN` (ideally `DESCENDING`) on your composite index inside the view — good.
- `TABLE ACCESS FULL` on the child table inside the view — **bad**: the probe scans everything per parent. Add/fix the index.
- For real runtime statistics, prefer `SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST'))` after running the query with the `/*+ GATHER_PLAN_STATISTICS */` hint — the `Starts` column shows exactly how many times the lateral view executed.

### 14.6 Common Reasons `APPLY` Queries Become Slow

| Cause | Symptom | Fix |
|---|---|---|
| Missing index on correlated column | Full scan of child table per parent | Composite index: `(correlation_col, order_by_col)` |
| Right side returns too many rows | Massive fan-out, huge intermediate result | Add `FETCH FIRST`, tighten filters inside the block |
| Filtering too late | Applied block computes everything, outer query discards most | Push predicates *into* the block (e.g. `status = 'SUCCESS'` inside, not outside) |
| `ORDER BY` without supporting index | `SORT ORDER BY` per parent row | Index whose trailing column matches the `ORDER BY` |
| Using `APPLY` where a simple join suffices | Nested loops forced where a hash join would fly | Rewrite as plain `JOIN`/`LEFT JOIN` |
| Expensive PL/SQL functions inside the block | CPU burn multiplied by parent count | Cache with scalar-subquery caching, `DETERMINISTIC`, or precompute |
| Applying over an unfiltered million-row parent | Millions of probes | Filter/paginate parents first; consider window functions for full-table reports |

---

## 15. Laravel Developer Section

### 15.1 Why Laravel Developers Should Care

Eloquent speaks a lowest-common-denominator SQL. When your Laravel app runs on Oracle (typically via [`yajra/laravel-oci8`](https://github.com/yajra/laravel-oci8)), the patterns that make dashboards fast — latest-row-per-parent, top-N-per-group, chained correlated lookups — have no first-class Eloquent equivalent. Knowing `APPLY` lets you replace N+1 query storms and PHP-side filtering with **one** optimized round trip.

Real Laravel use cases:

- **Customer dashboard** — one query instead of `$customer->orders->last()` per customer.
- **Latest order per customer / latest payment per order** — order lists, invoice screens.
- **Top products per order** — order detail pages, packing slips.
- **Admin reporting** — exports where every parent row must appear (`OUTER APPLY`).
- **API response optimization** — build the exact JSON shape in one query instead of nested lazy loads.

### 15.2 `DB::select()` With `OUTER APPLY` and Bind Parameters

```php
use Illuminate\Support\Facades\DB;

$rows = DB::select("
    SELECT c.customer_id,
           c.customer_name,
           lo.order_date    AS latest_order_date,
           lo.total_amount  AS latest_order_amount,
           NVL(lp.status, 'NO PAYMENT') AS last_payment_status
    FROM   customers c
    OUTER APPLY (
        SELECT o.order_id, o.order_date, o.total_amount
        FROM   orders o
        WHERE  o.customer_id = c.customer_id
        ORDER  BY o.order_date DESC, o.order_id DESC
        FETCH  FIRST 1 ROW ONLY
    ) lo
    OUTER APPLY (
        SELECT p.status
        FROM   payments p
        WHERE  p.order_id = lo.order_id
        ORDER  BY p.payment_date DESC, p.payment_id DESC
        FETCH  FIRST 1 ROW ONLY
    ) lp
    WHERE  c.country = :country          -- named bind parameter
    ORDER  BY c.customer_id
", ['country' => 'Bangladesh']);
```

Bind parameters (`:country`) are non-negotiable: they prevent SQL injection **and** let Oracle reuse the parsed cursor (soft parse) instead of hard-parsing a new statement per request.

### 15.3 Example Service Method

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\DB;

class CustomerDashboardService
{
    /**
     * Full dashboard row for one customer in a single Oracle round trip.
     */
    public function getCustomerDashboard(int $customerId): array
    {
        $sql = "
            SELECT c.customer_id,
                   c.customer_name,
                   lo.order_date                  AS latest_order_date,
                   lo.total_amount                AS latest_order_amount,
                   NVL(lp.status, 'NO PAYMENT')   AS last_payment_status,
                   NVL(lt.status, 'NO TICKETS')   AS last_ticket_status,
                   NVL(agg.total_order_value, 0)  AS total_order_value,
                   NVL(agg.order_count, 0)        AS number_of_orders
            FROM   customers c
            OUTER APPLY (
                SELECT o.order_id, o.order_date, o.total_amount
                FROM   orders o
                WHERE  o.customer_id = c.customer_id
                ORDER  BY o.order_date DESC, o.order_id DESC
                FETCH  FIRST 1 ROW ONLY
            ) lo
            OUTER APPLY (
                SELECT p.status
                FROM   payments p
                WHERE  p.order_id = lo.order_id
                ORDER  BY p.payment_date DESC, p.payment_id DESC
                FETCH  FIRST 1 ROW ONLY
            ) lp
            OUTER APPLY (
                SELECT t.status
                FROM   customer_support_tickets t
                WHERE  t.customer_id = c.customer_id
                ORDER  BY t.created_at DESC, t.ticket_id DESC
                FETCH  FIRST 1 ROW ONLY
            ) lt
            OUTER APPLY (
                SELECT SUM(o.total_amount) AS total_order_value,
                       COUNT(*)            AS order_count
                FROM   orders o
                WHERE  o.customer_id = c.customer_id
                AND    o.status <> 'CANCELLED'
            ) agg
            WHERE  c.customer_id = :customer_id
        ";

        $row = DB::selectOne($sql, ['customer_id' => $customerId]);

        return $row ? (array) $row : [];
    }
}
```

### 15.4 Why Raw SQL Beats Eloquent Here

- Eloquent's query builder has **no `APPLY`/`LATERAL` grammar**; you would be string-concatenating fragments through `selectRaw()`/`joinSub()` anyway — worst of both worlds.
- Advanced Oracle features (`FETCH FIRST`, `KEEP (DENSE_RANK ...)`, hints, lateral views) are only reachable through raw SQL.
- One explicit, reviewed, `EXPLAIN PLAN`-verified statement is easier to tune than an ORM-generated one. Keep raw SQL isolated in service/repository classes and hydrate results into DTOs — you keep Laravel's architecture and Oracle's power.

### 15.5 How `APPLY` Kills N+1 Problems

The classic Laravel N+1:

```php
// ❌ 1 query for customers + 1 query PER customer = N+1 round trips
$customers = Customer::all();
foreach ($customers as $customer) {
    $latest = $customer->orders()->latest('order_date')->first(); // one query each!
}
```

Even eager loading doesn't fully save you:

```php
// ⚠️ Better, but loads EVERY order of every customer into PHP memory,
// then discards all but one per customer
$customers = Customer::with('orders')->get();
$latest = $customers->map(fn ($c) => $c->orders->sortByDesc('order_date')->first());
```

The `OUTER APPLY` version is **one round trip**, and Oracle returns exactly one order per customer — nothing is over-fetched, nothing is filtered in PHP.

### 15.6 Common Laravel Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Running one query per customer in a loop | N+1 round trips, latency explodes | One `APPLY` query via `DB::select()` |
| Loading all orders and filtering in PHP | Memory bloat, slow collection ops | Let Oracle do `ORDER BY ... FETCH FIRST` |
| String-interpolating values into SQL | SQL injection + hard-parse storms | Always bind: `['id' => $id]` |
| Not indexing correlated columns | Each probe = full scan; page times out under load | Composite indexes (§14.3) |
| Eloquent loops instead of one optimized SQL | CPU + queries scale with data size | Service class + raw `APPLY` query |
| Confusing `OUTER APPLY` with `LEFT JOIN` | Fan-out double-counts sums in reports | `APPLY` a 1-row block; understand §11.2 |

---

## 16. Practical Real-Life Use Cases

The same two patterns — *latest row per parent* and *top-N per parent* — appear everywhere:

| Domain | Query | Operator |
|---|---|---|
| E-commerce | Latest order per customer | `OUTER APPLY` (dashboards) / `CROSS APPLY` (active buyers only) |
| Billing | Latest payment per invoice | `OUTER APPLY` (unpaid invoices must show) |
| E-commerce | Top-N products per order | `CROSS APPLY` + `FETCH FIRST n ROWS ONLY` |
| HR | Latest salary record per employee | `CROSS APPLY` (every employee has one) |
| CRM | Latest address per user | `OUTER APPLY` |
| Workflow | Latest approval status per request | `OUTER APPLY` (pending requests have none) |
| Security | Last login per user | `OUTER APPLY` (never-logged-in users matter) |
| Compliance | Most recent audit log per entity | `OUTER APPLY` on `audit_logs (entity_type, entity_id, logged_at)` |
| Reporting | Customer dashboard reports | Chained `OUTER APPLY` (§13) |
| Banking | Latest transaction per account | `OUTER APPLY` (dormant accounts must appear) |
| Education | Latest exam result per student | `OUTER APPLY` |
| Healthcare | Latest visit per patient | `OUTER APPLY` |
| Inventory | Latest stock movement per item | `OUTER APPLY` (items with no movement = audit targets) |

Notice the pattern: **`OUTER APPLY` dominates reporting** because "the rows with no match" (dormant accounts, unpaid invoices, never-sold products) are usually the *most* interesting rows in a business report.

---

## 17. Comparison Tables

### 17.1 `CROSS APPLY` vs `OUTER APPLY`

| Aspect | `CROSS APPLY` | `OUTER APPLY` |
|---|---|---|
| Left rows with no right-side match | **Dropped** | **Kept**, right columns `NULL` |
| Analogy | `INNER JOIN` + correlated view | `LEFT JOIN` + correlated view |
| Acts as a filter? | Yes | No |
| Typical use | "Only parents that have X" | Dashboards, reports, exports |
| Aggregate block without `GROUP BY` | Always returns 1 row → behaves like OUTER unless you add `HAVING COUNT(*) > 0` | Natural fit |

### 17.2 `CROSS APPLY` vs `INNER JOIN`

| Aspect | `CROSS APPLY` | `INNER JOIN` |
|---|---|---|
| Right side sees left columns | ✅ | ❌ (only in `ON` predicate) |
| Per-parent `ORDER BY` + `FETCH FIRST` | ✅ | ❌ |
| Per-parent aggregation in-place | ✅ | Needs derived table + `GROUP BY` |
| Optimizer join-method freedom | Often limited to nested loops | Hash/merge/nested — full freedom |
| Simple key matching | Overkill | ✅ Preferred |

### 17.3 `OUTER APPLY` vs `LEFT JOIN`

| Aspect | `OUTER APPLY` | `LEFT JOIN` |
|---|---|---|
| Preserves all left rows | ✅ | ✅ |
| "Latest child" without pre-ranking | ✅ | ❌ (needs `ROW_NUMBER()` derived table) |
| Fan-out risk with 1:N children | None if block returns ≤ 1 row | High — sums/counts corrupt |
| Multiple independent lookups per parent | Chain blocks cleanly | Joins interact; fan-out multiplies |
| Plain 1:N listing (all children) | Works but pointless | ✅ Preferred |

### 17.4 `APPLY` vs Correlated Subquery

| Aspect | `APPLY` | Correlated scalar subquery |
|---|---|---|
| Columns returned | Many | Exactly 1 |
| Rows returned per parent | Many (limitable) | At most 1 (else ORA-01427) |
| Reusable alias in outer query | ✅ | ❌ (repeat the subquery) |
| Reads well with 3+ values | ✅ | Becomes a wall of subqueries |
| Single quick lookup value | Fine | Fine (and benefits from scalar subquery caching) |

### 17.5 `APPLY` vs Window Functions

| Aspect | `APPLY` (+ `FETCH FIRST`) | `ROW_NUMBER()` window |
|---|---|---|
| Best for | Few/filtered parents, indexed probes | Full-table top-N reports |
| Access pattern | N indexed probes | 1 scan + window sort |
| Uses parent columns in child logic | ✅ | ❌ |
| Requires Oracle version | 12.1+ | 8i+ (windows), works everywhere |
| Rank value available in output | Only if computed inside block | ✅ Native |

### 17.6 `APPLY` vs `LATERAL`

| Aspect | `CROSS`/`OUTER APPLY` | `LATERAL` |
|---|---|---|
| Standard | Vendor extension (SQL Server heritage) | ANSI SQL standard |
| Oracle support | 12.1+ | 12.1+ |
| Also available in | SQL Server | PostgreSQL, MySQL 8+, DB2, Oracle |
| Outer semantics | `OUTER APPLY` (clean) | `LEFT JOIN LATERAL ... ON 1=1` (noisy) |
| Semantics | Identical | Identical |

### 17.7 Common Mistakes and Fixes

| Mistake | Fix |
|---|---|
| `COSS APPLY` typo (ORA-00933/900) | Spell it `CROSS APPLY` |
| Customers vanish from dashboard | You used `CROSS APPLY`; switch to `OUTER APPLY` |
| Report full of useless NULL rows | You used `OUTER APPLY`; switch to `CROSS APPLY` or filter |
| Query is slow at scale | Index `(correlation_col, order_by_col)`; check plan for full scans in the lateral view |
| `FETCH FIRST 1` returns different rows run-to-run | Add deterministic `ORDER BY` with a unique tie-breaker |
| Sums double-counted | You `LEFT JOIN`ed 1:N children; use a 1-row `OUTER APPLY` block |
| `TOP 1` / `OUTER APPLY ... ON` syntax errors | That's SQL Server muscle memory; Oracle uses `FETCH FIRST` and no `ON` clause |

---

## 18. Common Mistakes

1. **Writing `COSS APPLY` instead of `CROSS APPLY`.** Oracle raises a syntax error (`ORA-00933`/`ORA-00900`). Trivial, but it's the #1 "why doesn't APPLY work" question.
2. **Using `CROSS APPLY` when `OUTER APPLY` is needed.** The silent-row-loss bug: your dashboard "randomly" misses customers. If every left row must appear, it's `OUTER APPLY`, full stop.
3. **Using `OUTER APPLY` when unmatched rows are not required.** You pay for NULL-padded rows you then filter out (or worse, ship to the UI). If the match is mandatory, `CROSS APPLY` states the intent and lets the optimizer prune earlier.
4. **Forgetting the right side runs per left row.** An applied block that costs 50ms is fine for 20 rows and a disaster for 200,000. Always ask: *how many parents, and what does one probe cost?*
5. **Missing indexes on correlated columns.** The single biggest performance killer — every probe becomes a full scan. See §14.3.
6. **Returning too many rows from the applied subquery.** If you only need the latest row, `FETCH FIRST 1 ROW ONLY`; don't return 500 child rows and de-duplicate outside.
7. **Not using `ORDER BY` with `FETCH FIRST`.** Without `ORDER BY`, "first" means "whichever row Oracle reached first" — non-deterministic and plan-dependent. Also add a unique tie-breaker column.
8. **Assuming `APPLY` always outperforms joins.** For full-table top-N, a single-scan `ROW_NUMBER()` approach often wins. Measure.
9. **Using `APPLY` for simple joins.** `CROSS APPLY (SELECT ... WHERE key = key)` with no ordering/limits/aggregation is just an obfuscated, potentially slower `INNER JOIN`.
10. **Confusing Oracle syntax with SQL Server syntax.** In Oracle: `FETCH FIRST 1 ROW ONLY` (not `TOP 1`), no `ON` clause after `APPLY`, `NVL` instead of `ISNULL`, and remember `APPLY` needs 12c+.

---

## 19. Best Practices

- **Use `CROSS APPLY` when matching right-side rows are mandatory** — it filters and documents intent simultaneously.
- **Use `OUTER APPLY` when all left-side rows must be preserved** — dashboards, reports, exports, reconciliations.
- **Use `APPLY` for correlated top-N queries** — it is the most direct SQL expression of "top N children per parent".
- **Use `APPLY` when the right side needs left-side columns** beyond a simple equality — dynamic filters, `CASE` logic, chained lookups.
- **Use `FETCH FIRST 1 ROW ONLY` for latest-row queries** — pairs perfectly with a descending index range scan.
- **Always use deterministic `ORDER BY`** — include a unique tie-breaker (`order_id`) so "the latest" is stable across executions and plans.
- **Index correlated columns properly** — composite index on `(correlation_column, order_by_column)`; consider `DESC` and covering columns.
- **Keep the applied subquery small and selective** — filter inside the block, select only needed columns.
- **Compare with window functions** — for whole-table reports, benchmark the `ROW_NUMBER()` alternative.
- **Run `EXPLAIN PLAN` (and `DBMS_XPLAN.DISPLAY_CURSOR`) before shipping** any `APPLY` on large tables; verify index range scans inside the lateral view.
- **Use bind parameters in Laravel** — security and cursor reuse.
- **Don't use `APPLY` where a simple join is clearer** — clarity is a feature; save `APPLY` for the problems it uniquely solves.

---

## 20. Conclusion

`CROSS APPLY` and `OUTER APPLY` bring true **lateral correlation** to Oracle's `FROM` clause: the right-side query becomes a per-row function of the left side, free to use `ORDER BY`, `FETCH FIRST`, aggregation, joins, and multiple output columns.

The mental model fits in two lines:

> **`CROSS APPLY` = `INNER JOIN` a correlated inline view — parents without matches disappear.**
> **`OUTER APPLY` = `LEFT JOIN` a correlated inline view — every parent survives, gaps become `NULL`.**

Reach for them when you need *latest-row-per-parent*, *top-N-per-group*, *multi-column correlated lookups*, or *chained per-row logic* — and back them with composite indexes on `(correlation column, order column)` so each probe is a single index range scan. For whole-table analytics, keep `ROW_NUMBER()` in your toolbox and let `EXPLAIN PLAN` referee.

For Laravel developers on Oracle, one well-indexed `OUTER APPLY` statement routinely replaces an N+1 storm of Eloquent queries — the difference between a dashboard that times out and one that renders instantly.

*Oracle 12c Release 1 (12.1) or later required for `CROSS APPLY`, `OUTER APPLY`, `LATERAL`, and `FETCH FIRST`.*
