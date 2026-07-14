# Oracle Transaction Management — Advanced Guide

> A comprehensive, practical reference for database developers and Laravel engineers working with Oracle Database.

---

## Table of Contents

1. [Introduction to Oracle Transactions](#1-introduction-to-oracle-transactions)
2. [Why Transactions Matter in Real-World Applications](#2-why-transactions-matter-in-real-world-applications)
3. [ACID Properties](#3-acid-properties)
4. [Transaction Control Commands](#4-transaction-control-commands)
5. [When a Transaction Starts and Ends in Oracle](#5-when-a-transaction-starts-and-ends-in-oracle)
6. [Implicit Commit — DDL Commands](#6-implicit-commit--ddl-commands)
7. [Key Differences You Must Know](#7-key-differences-you-must-know)
8. [Oracle Locking and Concurrency](#8-oracle-locking-and-concurrency)
9. [Oracle Isolation Levels](#9-oracle-isolation-levels)
10. [Common Oracle Transaction Errors](#10-common-oracle-transaction-errors)
11. [Advanced Working Example — Bank Fund Transfer System](#11-advanced-working-example--bank-fund-transfer-system)
12. [Laravel Developer Section](#12-laravel-developer-section)
13. [Practical Real-Life Use Cases](#13-practical-real-life-use-cases)
14. [Best Practices](#14-best-practices)
15. [Common Mistakes](#15-common-mistakes)
16. [Conclusion](#16-conclusion)

---

## 1. Introduction to Oracle Transactions

A **transaction** in Oracle is a logical unit of work composed of one or more SQL statements that are treated as a single, indivisible operation. Either all statements in the transaction succeed and their effects are permanently saved, or none of them are applied — leaving the database in its prior consistent state.

Oracle uses a sophisticated **multi-version concurrency control (MVCC)** engine, which means readers never block writers and writers never block readers. This is fundamentally different from databases like MySQL (InnoDB) and gives Oracle exceptionally powerful concurrency characteristics.

```sql
-- A simple transaction
UPDATE accounts SET balance = balance - 1000 WHERE account_id = 101;
UPDATE accounts SET balance = balance + 1000 WHERE account_id = 202;
COMMIT;
```

Without transaction management, two simultaneous bank transfers could corrupt balances, leave funds stuck in limbo, or create phantom records. Transactions prevent all of that.

---

## 2. Why Transactions Matter in Real-World Applications

| Scenario | Without Transactions | With Transactions |
|---|---|---|
| Bank transfer | Debit succeeds, credit crashes → money vanishes | Both succeed or both roll back |
| E-commerce order | Stock deducted, payment fails → oversell | All steps atomic |
| Payroll processing | Some employees paid twice, some not at all | All-or-nothing payroll run |
| Hospital billing | Procedure recorded, invoice missing | Billing and procedure linked |
| Loan approval | Approval logged, disbursement skipped | Full workflow or full reversal |

Real-world applications involve **chains of dependent operations**. A transaction ensures that a chain either completes fully or is entirely undone, protecting data integrity at every step.

---

## 3. ACID Properties

Oracle fully supports all four ACID properties, which are the foundation of reliable transaction processing.

### Atomicity

> "All or nothing."

Every operation within a transaction is treated as a single unit. If any statement fails, the entire transaction is rolled back automatically.

```sql
BEGIN
  UPDATE accounts SET balance = balance - 500 WHERE account_id = 1;
  -- If this next line throws an error, the UPDATE above is rolled back
  UPDATE accounts SET balance = balance + 500 WHERE account_id = 999; -- non-existent
  COMMIT;
EXCEPTION
  WHEN OTHERS THEN
    ROLLBACK;
    RAISE;
END;
```

### Consistency

> "The database moves from one valid state to another valid state."

Constraints (primary keys, foreign keys, check constraints, NOT NULL) are enforced at statement or transaction level. Oracle will never allow a COMMIT that violates declared constraints.

```sql
-- This will fail because of a CHECK CONSTRAINT — balance cannot go negative
UPDATE accounts SET balance = -500 WHERE account_id = 1;
-- ORA-02290: check constraint violated
```

### Isolation

> "Concurrent transactions do not interfere with each other."

Oracle achieves isolation through MVCC (Multi-Version Consistency Control). Each session sees a consistent snapshot of the data as it existed at the start of the query (or transaction, depending on isolation level). A transaction's uncommitted changes are **invisible** to other sessions.

```sql
-- Session A (not committed yet)
UPDATE accounts SET balance = 9999 WHERE account_id = 1;

-- Session B (reads the original value — not 9999)
SELECT balance FROM accounts WHERE account_id = 1;
-- Returns original value, not 9999
```

### Durability

> "Once committed, changes survive failures."

After a `COMMIT`, Oracle writes the change to the **redo log** on disk before acknowledging success. Even if the server crashes immediately after, Oracle will recover all committed data on restart using the redo log.

---

## 4. Transaction Control Commands

### `COMMIT`

Permanently saves all changes made since the last COMMIT or ROLLBACK. Releases all row-level locks held by the session.

```sql
UPDATE employees SET salary = salary * 1.10 WHERE department_id = 50;
COMMIT;
-- Changes are now permanent and visible to all sessions
```

### `ROLLBACK`

Undoes all changes made since the last COMMIT or ROLLBACK. Releases all locks. The database returns to its last committed state.

```sql
UPDATE employees SET salary = 0; -- Oops, wrong update!
ROLLBACK;
-- All salary changes are reversed
```

### `SAVEPOINT`

Creates a named intermediate checkpoint within a transaction. You can roll back to this point without discarding the entire transaction.

```sql
INSERT INTO orders (order_id, customer_id) VALUES (1001, 55);
SAVEPOINT after_order_insert;

INSERT INTO order_items (order_id, product_id, qty) VALUES (1001, 99, 2);
SAVEPOINT after_items_insert;

UPDATE products SET stock = stock - 2 WHERE product_id = 99;
-- If stock update fails, roll back only the stock update
ROLLBACK TO SAVEPOINT after_items_insert;

COMMIT; -- Commits only the order header and items
```

### `ROLLBACK TO SAVEPOINT`

Undoes all changes made **after** the named savepoint, but keeps all changes made **before** it. The transaction remains open — you can continue working.

```sql
SAVEPOINT sp1;
UPDATE accounts SET balance = balance - 200 WHERE account_id = 1;

SAVEPOINT sp2;
UPDATE accounts SET balance = balance + 200 WHERE account_id = 2;

-- Something went wrong with account 2
ROLLBACK TO SAVEPOINT sp1;
-- Account 1 debit is also reversed, back to sp1 state

COMMIT; -- Commits nothing meaningful (or whatever was before sp1)
```

### `SET TRANSACTION`

Configures properties of the current transaction **before any DML** is executed. Must be the first statement in a transaction.

```sql
-- Set isolation level to SERIALIZABLE
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Set transaction as READ ONLY (no DML allowed)
SET TRANSACTION READ ONLY;

-- Name a transaction for tracing
SET TRANSACTION NAME 'fund_transfer_tx_001';
```

---

## 5. When a Transaction Starts and Ends in Oracle

### Transaction Start

A new transaction begins **implicitly** with the first DML statement (`INSERT`, `UPDATE`, `DELETE`, `MERGE`) after:
- A session connects to Oracle
- A previous transaction ends (via `COMMIT` or `ROLLBACK`)

Oracle does **not** require an explicit `BEGIN TRANSACTION` statement.

```sql
-- Transaction implicitly starts here
INSERT INTO audit_logs (event, created_at) VALUES ('Login', SYSDATE);

-- Transaction is still open...
UPDATE sessions SET last_seen = SYSDATE WHERE user_id = 10;

-- Transaction ends here
COMMIT;
```

### Transaction End

A transaction ends when:

| Cause | Effect |
|---|---|
| `COMMIT` | All changes made permanent |
| `ROLLBACK` | All changes reversed |
| DDL statement (`CREATE`, `ALTER`, `DROP`, `TRUNCATE`) | Implicit COMMIT before and after DDL |
| Session disconnect normally | Implicit COMMIT |
| Session crash / forced disconnect | Implicit ROLLBACK |
| Program exits without COMMIT | Implicit ROLLBACK |

---

## 6. Implicit Commit — DDL Commands

This is one of the most important — and dangerous — behaviors in Oracle. **DDL commands always trigger an implicit COMMIT** before and after they execute.

This means:

```sql
-- Step 1: DML (transaction open)
UPDATE accounts SET balance = balance - 500 WHERE account_id = 1;

-- Step 2: DDL — triggers IMPLICIT COMMIT of Step 1 first!
CREATE TABLE temp_log (id NUMBER); 
-- The UPDATE above is now permanently committed, even though you never typed COMMIT

-- Step 3: Even if you ROLLBACK now, Step 1 is already committed
ROLLBACK; -- Only rolls back changes after the DDL — nothing here
```

### DDL Commands That Trigger Implicit COMMIT

| Command | Implicit COMMIT |
|---|---|
| `CREATE TABLE` | Yes — before and after |
| `ALTER TABLE` | Yes — before and after |
| `DROP TABLE` | Yes — before and after |
| `TRUNCATE TABLE` | Yes — before and after |
| `CREATE INDEX` | Yes — before and after |
| `GRANT` / `REVOKE` | Yes |
| `CREATE SEQUENCE` | Yes |

> ⚠️ **Critical Rule:** Never mix DML and DDL inside the same logical transaction if you need the ability to roll back the DML. DDL will auto-commit it.

---

## 7. Key Differences You Must Know

### DML vs DDL Transaction Behavior

| Feature | DML (`INSERT`, `UPDATE`, `DELETE`) | DDL (`CREATE`, `ALTER`, `DROP`, `TRUNCATE`) |
|---|---|---|
| Starts transaction | Yes (implicitly) | No — ends current one first |
| Can be rolled back | Yes | No |
| Requires explicit COMMIT | Yes | No — auto-commits |
| Affects undo segments | Yes | Minimal (for DDL itself) |
| Triggers implicit COMMIT | No | Yes (before and after) |

### `DELETE` vs `TRUNCATE`

| Feature | `DELETE` | `TRUNCATE` |
|---|---|---|
| Type | DML | DDL |
| Can be rolled back | Yes | No |
| Fires triggers | Yes (`BEFORE`/`AFTER DELETE`) | No |
| WHERE clause | Yes | No |
| Speed | Slower (logs each row) | Much faster (deallocates extents) |
| Implicit COMMIT | No | Yes |
| Reclaims space | Only with manual rebuild | Yes, immediately |

```sql
-- DELETE: Can be rolled back
DELETE FROM temp_data WHERE created_at < SYSDATE - 30;
ROLLBACK; -- Works! All rows restored

-- TRUNCATE: Cannot be rolled back
TRUNCATE TABLE temp_data;
ROLLBACK; -- Does nothing — already auto-committed
```

### `ROLLBACK` vs `ROLLBACK TO SAVEPOINT`

| Feature | `ROLLBACK` | `ROLLBACK TO SAVEPOINT` |
|---|---|---|
| Scope | Entire transaction | Only changes after named savepoint |
| Transaction status | Ends the transaction | Transaction remains open |
| Locks released | All | Only for changes after savepoint |
| COMMIT needed after | No — transaction ended | Yes — to commit remaining work |

### Auto-commit vs Manual Commit

| Feature | Auto-commit (SQL*Plus `SET AUTOCOMMIT ON` or JDBC) | Manual Commit |
|---|---|---|
| When committed | After every single statement | Only when you call `COMMIT` |
| Rollback possible | No | Yes, until COMMIT issued |
| Risk | Cannot undo mistakes | More control, but must remember to COMMIT |
| Use case | Quick scripts, reporting | Production transactional code |

---

## 8. Oracle Locking and Concurrency

### Row-Level Locking

Oracle locks only the specific rows being modified — not the entire table. This allows maximum concurrency. Other sessions can read the same table freely and can even modify different rows simultaneously.

```sql
-- Session A locks row where account_id = 1
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
-- Row 1 is locked until Session A COMMITs or ROLLBACKs

-- Session B can still modify a different row freely
UPDATE accounts SET balance = balance + 100 WHERE account_id = 2; -- No conflict
```

### `SELECT ... FOR UPDATE`

Explicitly locks rows returned by a SELECT before modifying them. This prevents another session from modifying those rows between your SELECT and your UPDATE — eliminating the **lost update problem**.

```sql
-- Lock the row first, then read it
SELECT balance
FROM accounts
WHERE account_id = 1
FOR UPDATE;

-- Now safely update — no one else can change this row
UPDATE accounts SET balance = balance - 500 WHERE account_id = 1;
COMMIT;
```

Use `FOR UPDATE NOWAIT` to fail immediately if the row is already locked:

```sql
SELECT balance FROM accounts WHERE account_id = 1
FOR UPDATE NOWAIT;
-- Raises ORA-00054 if another session holds the lock
```

Use `FOR UPDATE WAIT n` to wait up to `n` seconds:

```sql
SELECT balance FROM accounts WHERE account_id = 1
FOR UPDATE WAIT 5;
-- Raises ORA-30006 if lock not acquired within 5 seconds
```

### Blocking Sessions

When Session A locks a row and Session B tries to modify the same row, Session B **blocks** (waits). It does not get an error immediately — it simply hangs until Session A commits or rolls back.

```sql
-- Identify blocking sessions
SELECT
    blocking_session,
    sid,
    serial#,
    wait_class,
    seconds_in_wait,
    event
FROM v$session
WHERE blocking_session IS NOT NULL;
```

### Deadlocks

A **deadlock** occurs when two sessions each hold a lock that the other session is waiting for. Oracle detects deadlocks automatically within seconds and resolves them by rolling back **one statement** (not the whole transaction) of one of the sessions with:

```
ORA-00060: deadlock detected while waiting for resource
```

```
-- Session A                              -- Session B
UPDATE accounts SET ...                  UPDATE accounts SET ...
  WHERE account_id = 1; -- Lock A1         WHERE account_id = 2; -- Lock B2

UPDATE accounts SET ...                  UPDATE accounts SET ...
  WHERE account_id = 2; -- Waits for B2    WHERE account_id = 1; -- Waits for A1
-- DEADLOCK! Oracle rolls back one statement
```

**Prevention:** Always lock multiple rows in the **same order** across all sessions/transactions.

### Lost Update Problem

Without `SELECT ... FOR UPDATE`, two sessions can read the same value simultaneously, both compute an update based on the old value, and one update overwrites the other.

```sql
-- Session A reads balance = 1000
-- Session B reads balance = 1000
-- Session A updates: balance = 1000 - 200 = 800, commits
-- Session B updates: balance = 1000 + 500 = 1500, commits
-- Final balance: 1500 — Session A's debit is LOST!

-- Correct approach using FOR UPDATE:
SELECT balance FROM accounts WHERE account_id = 1 FOR UPDATE;
-- Now Session B must wait until Session A commits
```

---

## 9. Oracle Isolation Levels

### `READ COMMITTED` (Default)

Each statement in the transaction sees only data that was **committed before that statement began**. This is Oracle's default and provides excellent concurrency with good data consistency for most use cases.

- **Allows:** Non-repeatable reads (the same row can return different values if read twice in one transaction)
- **Prevents:** Dirty reads (you never see another session's uncommitted data)

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED; -- Oracle default
SELECT balance FROM accounts WHERE account_id = 1;
-- Returns committed balance at this exact moment
```

### `SERIALIZABLE`

The entire transaction sees only data that was committed **before the transaction began**. No phantom reads, no non-repeatable reads. Provides the highest isolation. However, if another session modifies a row you've already read, your transaction will fail with `ORA-08177`.

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

SELECT balance FROM accounts WHERE account_id = 1; -- Reads 1000

-- Session B commits an update: balance = 1500

SELECT balance FROM accounts WHERE account_id = 1; -- Still reads 1000 (snapshot!)

UPDATE accounts SET balance = balance - 200 WHERE account_id = 1;
-- If Session B modified this row — ORA-08177: can't serialize access
```

### `READ ONLY`

The transaction can only issue SELECT statements. No DML is permitted. Guarantees a perfectly consistent read snapshot of the entire database for the duration of the transaction — ideal for long-running reports.

```sql
SET TRANSACTION READ ONLY;

SELECT SUM(balance) FROM accounts; -- Consistent snapshot
SELECT COUNT(*) FROM transactions WHERE created_at > SYSDATE - 1; -- Same snapshot
-- No INSERT/UPDATE/DELETE allowed until COMMIT or ROLLBACK
```

### Isolation Level Comparison

| Feature | READ COMMITTED | SERIALIZABLE | READ ONLY |
|---|---|---|---|
| Dirty reads | Prevented | Prevented | Prevented |
| Non-repeatable reads | Possible | Prevented | Prevented |
| Phantom reads | Possible | Prevented | Prevented |
| DML allowed | Yes | Yes | No |
| ORA-08177 risk | No | Yes | No |
| Best for | OLTP | Financial integrity | Reporting |

---

## 10. Common Oracle Transaction Errors

### `ORA-00060: deadlock detected while waiting for resource`

Oracle detected a circular lock dependency and automatically rolled back one statement.

**Fix:** Handle `ORA-00060` in your exception block, ROLLBACK the full transaction, and retry. Always lock rows in a consistent, deterministic order.

```sql
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE = -60 THEN
      ROLLBACK;
      -- log the deadlock and retry logic here
    END IF;
```

### `ORA-08177: can't serialize access for this transaction`

Raised under `SERIALIZABLE` isolation when another session has modified a row that your transaction has already read. Your transaction cannot be safely serialized.

**Fix:** Catch this error, ROLLBACK, and retry the entire transaction.

```sql
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE = -8177 THEN
      ROLLBACK;
      -- Retry the full serializable transaction
    END IF;
```

### `ORA-01555: snapshot too old`

Oracle's undo segments store the "before image" of data for MVCC. If a long-running query or transaction needs an old snapshot, but Oracle has already overwritten those undo segments (due to insufficient UNDO tablespace or very long queries), this error is raised.

**Fix:** Increase `UNDO_RETENTION` parameter, increase UNDO tablespace size, or optimize long-running queries to complete faster.

```sql
-- Check current undo retention (in seconds)
SELECT value FROM v$parameter WHERE name = 'undo_retention';

-- Increase it (DBA privilege required)
ALTER SYSTEM SET UNDO_RETENTION = 3600; -- 1 hour
```

### `ORA-02291: integrity constraint violated - parent key not found`

You tried to insert a child record referencing a parent that does not exist.

```sql
-- accounts table has no account_id = 9999
INSERT INTO transactions (txn_id, account_id, amount)
VALUES (1, 9999, 500);
-- ORA-02291: integrity constraint (FK_TXN_ACCOUNT) violated - parent key not found
```

**Fix:** Always ensure the parent record exists before inserting children, or insert the parent first within the same transaction.

### `ORA-02292: integrity constraint violated - child record found`

You tried to delete or update a parent record that still has dependent child records.

```sql
-- transactions table references account_id = 1
DELETE FROM accounts WHERE account_id = 1;
-- ORA-02292: integrity constraint (FK_TXN_ACCOUNT) violated - child record found
```

**Fix:** Delete child records first, or use `ON DELETE CASCADE` on the foreign key constraint (if appropriate for your data model).

---

## 11. Advanced Working Example — Bank Fund Transfer System

This is a complete, realistic Oracle SQL and PL/SQL implementation of a bank fund transfer system demonstrating every concept covered above.

### 11.1 Table Design

```sql
-- ============================================================
-- BANK FUND TRANSFER SYSTEM — COMPLETE TABLE DEFINITIONS
-- ============================================================

-- Drop tables in correct dependency order (if they exist)
BEGIN
  EXECUTE IMMEDIATE 'DROP TABLE audit_logs CASCADE CONSTRAINTS';
  EXECUTE IMMEDIATE 'DROP TABLE transactions CASCADE CONSTRAINTS';
  EXECUTE IMMEDIATE 'DROP TABLE accounts CASCADE CONSTRAINTS';
  EXECUTE IMMEDIATE 'DROP TABLE customers CASCADE CONSTRAINTS';
EXCEPTION
  WHEN OTHERS THEN NULL; -- Ignore errors if tables don't exist
END;
/

-- -------------------------------------------------------
-- CUSTOMERS TABLE
-- -------------------------------------------------------
CREATE TABLE customers (
    customer_id   NUMBER(10)      GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    full_name     VARCHAR2(100)   NOT NULL,
    email         VARCHAR2(150)   UNIQUE NOT NULL,
    phone         VARCHAR2(20),
    status        VARCHAR2(20)    DEFAULT 'ACTIVE'
                                  CHECK (status IN ('ACTIVE', 'SUSPENDED', 'CLOSED')),
    created_at    TIMESTAMP       DEFAULT SYSTIMESTAMP NOT NULL
);

-- -------------------------------------------------------
-- ACCOUNTS TABLE
-- -------------------------------------------------------
CREATE TABLE accounts (
    account_id     NUMBER(10)      GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id    NUMBER(10)      NOT NULL,
    account_number VARCHAR2(20)    UNIQUE NOT NULL,
    account_type   VARCHAR2(20)    DEFAULT 'SAVINGS'
                                   CHECK (account_type IN ('SAVINGS', 'CURRENT', 'FIXED')),
    balance        NUMBER(15, 2)   DEFAULT 0.00 NOT NULL,
    currency       VARCHAR2(3)     DEFAULT 'USD' NOT NULL,
    status         VARCHAR2(20)    DEFAULT 'ACTIVE'
                                   CHECK (status IN ('ACTIVE', 'FROZEN', 'CLOSED')),
    minimum_balance NUMBER(10, 2)  DEFAULT 100.00 NOT NULL,
    created_at     TIMESTAMP       DEFAULT SYSTIMESTAMP NOT NULL,
    CONSTRAINT fk_acc_customer FOREIGN KEY (customer_id)
        REFERENCES customers(customer_id),
    CONSTRAINT chk_balance_positive CHECK (balance >= 0)
);

-- -------------------------------------------------------
-- TRANSACTIONS TABLE
-- -------------------------------------------------------
CREATE TABLE transactions (
    txn_id          NUMBER(15)     GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    txn_reference   VARCHAR2(50)   UNIQUE NOT NULL,
    txn_type        VARCHAR2(30)   NOT NULL
                                   CHECK (txn_type IN ('TRANSFER', 'DEPOSIT',
                                                        'WITHDRAWAL', 'FEE', 'REVERSAL')),
    from_account_id NUMBER(10),
    to_account_id   NUMBER(10),
    amount          NUMBER(15, 2)  NOT NULL,
    fee_amount      NUMBER(10, 2)  DEFAULT 0.00,
    currency        VARCHAR2(3)    DEFAULT 'USD',
    status          VARCHAR2(20)   DEFAULT 'PENDING'
                                   CHECK (status IN ('PENDING', 'COMPLETED',
                                                     'FAILED', 'REVERSED')),
    description     VARCHAR2(500),
    created_at      TIMESTAMP      DEFAULT SYSTIMESTAMP NOT NULL,
    completed_at    TIMESTAMP,
    CONSTRAINT fk_txn_from_acc FOREIGN KEY (from_account_id)
        REFERENCES accounts(account_id),
    CONSTRAINT fk_txn_to_acc FOREIGN KEY (to_account_id)
        REFERENCES accounts(account_id),
    CONSTRAINT chk_txn_amount CHECK (amount > 0)
);

-- -------------------------------------------------------
-- AUDIT LOGS TABLE
-- -------------------------------------------------------
CREATE TABLE audit_logs (
    log_id       NUMBER(15)    GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    event_type   VARCHAR2(50)  NOT NULL,
    table_name   VARCHAR2(50),
    record_id    NUMBER(15),
    old_value    VARCHAR2(4000),
    new_value    VARCHAR2(4000),
    description  VARCHAR2(1000),
    performed_by VARCHAR2(100) DEFAULT USER,
    created_at   TIMESTAMP     DEFAULT SYSTIMESTAMP NOT NULL
);

-- -------------------------------------------------------
-- INDEXES FOR PERFORMANCE
-- -------------------------------------------------------
CREATE INDEX idx_acc_customer     ON accounts(customer_id);
CREATE INDEX idx_txn_from_acc     ON transactions(from_account_id);
CREATE INDEX idx_txn_to_acc       ON transactions(to_account_id);
CREATE INDEX idx_txn_reference    ON transactions(txn_reference);
CREATE INDEX idx_audit_event      ON audit_logs(event_type, created_at);
```

### 11.2 Sample Data

```sql
-- -------------------------------------------------------
-- INSERT SAMPLE CUSTOMERS
-- -------------------------------------------------------
INSERT INTO customers (full_name, email, phone, status)
VALUES ('Alice Johnson', 'alice@bank.com', '+1-555-0101', 'ACTIVE');

INSERT INTO customers (full_name, email, phone, status)
VALUES ('Bob Martinez', 'bob@bank.com', '+1-555-0202', 'ACTIVE');

INSERT INTO customers (full_name, email, phone, status)
VALUES ('Carol Williams', 'carol@bank.com', '+1-555-0303', 'SUSPENDED');

COMMIT;

-- -------------------------------------------------------
-- INSERT SAMPLE ACCOUNTS
-- -------------------------------------------------------
INSERT INTO accounts (customer_id, account_number, account_type, balance, currency, status, minimum_balance)
VALUES (1, 'ACC-001-ALICE', 'SAVINGS', 5000.00, 'USD', 'ACTIVE', 100.00);

INSERT INTO accounts (customer_id, account_number, account_type, balance, currency, status, minimum_balance)
VALUES (2, 'ACC-002-BOB', 'SAVINGS', 2000.00, 'USD', 'ACTIVE', 100.00);

INSERT INTO accounts (customer_id, account_number, account_type, balance, currency, status, minimum_balance)
VALUES (3, 'ACC-003-CAROL', 'CURRENT', 500.00, 'USD', 'ACTIVE', 50.00);

COMMIT;
```

### 11.3 Before-Transaction State

```sql
-- ================================================
-- CHECK BALANCES BEFORE TRANSACTION
-- ================================================
SELECT
    a.account_id,
    a.account_number,
    c.full_name,
    a.balance,
    a.status
FROM accounts a
JOIN customers c ON c.customer_id = a.customer_id
ORDER BY a.account_id;
```

**Expected Output (Before):**

| ACCOUNT_ID | ACCOUNT_NUMBER | FULL_NAME | BALANCE | STATUS |
|---|---|---|---|---|
| 1 | ACC-001-ALICE | Alice Johnson | 5000.00 | ACTIVE |
| 2 | ACC-002-BOB | Bob Martinez | 2000.00 | ACTIVE |
| 3 | ACC-003-CAROL | Carol Williams | 500.00 | ACTIVE |

### 11.4 The Fund Transfer Stored Procedure

```sql
-- ============================================================
-- STORED PROCEDURE: transfer_funds
-- Transfers money between two accounts with full transaction
-- management, row locking, savepoints, and error handling.
-- ============================================================
CREATE OR REPLACE PROCEDURE transfer_funds (
    p_from_account_id  IN  NUMBER,
    p_to_account_id    IN  NUMBER,
    p_amount           IN  NUMBER,
    p_description      IN  VARCHAR2 DEFAULT NULL,
    p_txn_fee          IN  NUMBER   DEFAULT 2.50,
    p_txn_reference    OUT VARCHAR2,
    p_status           OUT VARCHAR2,
    p_message          OUT VARCHAR2
)
AS
    -- --------------------------------------------------------
    -- Local variables
    -- --------------------------------------------------------
    v_from_balance       accounts.balance%TYPE;
    v_to_balance         accounts.balance%TYPE;
    v_from_status        accounts.status%TYPE;
    v_to_status          accounts.status%TYPE;
    v_from_customer      customers.status%TYPE;
    v_to_customer        customers.status%TYPE;
    v_min_balance        accounts.minimum_balance%TYPE;
    v_txn_id             NUMBER;
    v_total_debit        NUMBER;

    -- --------------------------------------------------------
    -- Custom exceptions
    -- --------------------------------------------------------
    ex_invalid_amount    EXCEPTION;
    ex_same_account      EXCEPTION;
    ex_account_frozen    EXCEPTION;
    ex_customer_suspended EXCEPTION;
    ex_insufficient_funds EXCEPTION;
    ex_account_not_found EXCEPTION;
    ex_invalid_fee       EXCEPTION;

BEGIN
    -- --------------------------------------------------------
    -- Step 1: Generate unique transaction reference
    -- --------------------------------------------------------
    p_txn_reference := 'TXN-' || TO_CHAR(SYSTIMESTAMP, 'YYYYMMDDHH24MISSFF3')
                        || '-' || p_from_account_id || '-' || p_to_account_id;

    -- --------------------------------------------------------
    -- Step 2: Basic input validation
    -- --------------------------------------------------------
    IF p_amount <= 0 THEN
        RAISE ex_invalid_amount;
    END IF;

    IF p_txn_fee < 0 THEN
        RAISE ex_invalid_fee;
    END IF;

    IF p_from_account_id = p_to_account_id THEN
        RAISE ex_same_account;
    END IF;

    v_total_debit := p_amount + p_txn_fee;

    -- --------------------------------------------------------
    -- SAVEPOINT 1: Before any database changes
    -- --------------------------------------------------------
    SAVEPOINT sp_before_transfer;

    -- --------------------------------------------------------
    -- Step 3: Lock source account row (SELECT FOR UPDATE)
    -- Prevents lost update problem — no other session can
    -- modify this row until we COMMIT or ROLLBACK.
    -- Lock both rows in account_id order to prevent deadlocks.
    -- --------------------------------------------------------
    BEGIN
        IF p_from_account_id < p_to_account_id THEN
            -- Lock from-account first (lower ID)
            SELECT a.balance, a.status, a.minimum_balance, c.status
            INTO v_from_balance, v_from_status, v_min_balance, v_from_customer
            FROM accounts a
            JOIN customers c ON c.customer_id = a.customer_id
            WHERE a.account_id = p_from_account_id
            FOR UPDATE WAIT 10; -- Wait up to 10 seconds for lock

            -- Then lock to-account
            SELECT a.balance, a.status, c.status
            INTO v_to_balance, v_to_status, v_to_customer
            FROM accounts a
            JOIN customers c ON c.customer_id = a.customer_id
            WHERE a.account_id = p_to_account_id
            FOR UPDATE WAIT 10;
        ELSE
            -- Lock to-account first (lower ID)
            SELECT a.balance, a.status, c.status
            INTO v_to_balance, v_to_status, v_to_customer
            FROM accounts a
            JOIN customers c ON c.customer_id = a.customer_id
            WHERE a.account_id = p_to_account_id
            FOR UPDATE WAIT 10;

            -- Then lock from-account
            SELECT a.balance, a.status, a.minimum_balance, c.status
            INTO v_from_balance, v_from_status, v_min_balance, v_from_customer
            FROM accounts a
            JOIN customers c ON c.customer_id = a.customer_id
            WHERE a.account_id = p_from_account_id
            FOR UPDATE WAIT 10;
        END IF;
    EXCEPTION
        WHEN NO_DATA_FOUND THEN
            RAISE ex_account_not_found;
    END;

    -- --------------------------------------------------------
    -- Step 4: Business rule validations (after locking rows)
    -- --------------------------------------------------------
    IF v_from_status != 'ACTIVE' THEN
        RAISE ex_account_frozen;
    END IF;

    IF v_to_status != 'ACTIVE' THEN
        RAISE ex_account_frozen;
    END IF;

    IF v_from_customer = 'SUSPENDED' THEN
        RAISE ex_customer_suspended;
    END IF;

    IF (v_from_balance - v_total_debit) < v_min_balance THEN
        RAISE ex_insufficient_funds;
    END IF;

    -- --------------------------------------------------------
    -- Step 5: Create the PENDING transaction record
    -- --------------------------------------------------------
    INSERT INTO transactions (
        txn_reference, txn_type,
        from_account_id, to_account_id,
        amount, fee_amount, currency,
        status, description
    )
    VALUES (
        p_txn_reference, 'TRANSFER',
        p_from_account_id, p_to_account_id,
        p_amount, p_txn_fee, 'USD',
        'PENDING', NVL(p_description, 'Fund Transfer')
    )
    RETURNING txn_id INTO v_txn_id;

    -- --------------------------------------------------------
    -- SAVEPOINT 2: Transaction record created
    -- --------------------------------------------------------
    SAVEPOINT sp_txn_recorded;

    -- --------------------------------------------------------
    -- Step 6: Audit log — transfer initiated
    -- --------------------------------------------------------
    INSERT INTO audit_logs (event_type, table_name, record_id,
                             old_value, new_value, description)
    VALUES ('TRANSFER_INITIATED', 'transactions', v_txn_id,
            NULL,
            'Amount: ' || p_amount || ', Fee: ' || p_txn_fee,
            'Transfer from account ' || p_from_account_id
            || ' to ' || p_to_account_id);

    -- --------------------------------------------------------
    -- SAVEPOINT 3: Audit log written, about to debit
    -- --------------------------------------------------------
    SAVEPOINT sp_audit_initiated;

    -- --------------------------------------------------------
    -- Step 7: Debit source account (amount + fee)
    -- --------------------------------------------------------
    UPDATE accounts
    SET    balance    = balance - v_total_debit
    WHERE  account_id = p_from_account_id;

    IF SQL%ROWCOUNT = 0 THEN
        -- Should not happen since we locked, but guard anyway
        ROLLBACK TO SAVEPOINT sp_before_transfer;
        RAISE ex_account_not_found;
    END IF;

    -- --------------------------------------------------------
    -- SAVEPOINT 4: Source debited successfully
    -- --------------------------------------------------------
    SAVEPOINT sp_debited;

    -- --------------------------------------------------------
    -- Step 8: Credit destination account
    -- --------------------------------------------------------
    UPDATE accounts
    SET    balance    = balance + p_amount
    WHERE  account_id = p_to_account_id;

    IF SQL%ROWCOUNT = 0 THEN
        -- Destination update failed — roll back debit
        ROLLBACK TO SAVEPOINT sp_audit_initiated;
        -- Update transaction status to FAILED
        UPDATE transactions
        SET    status       = 'FAILED',
               completed_at = SYSTIMESTAMP
        WHERE  txn_id = v_txn_id;

        INSERT INTO audit_logs (event_type, table_name, record_id, description)
        VALUES ('TRANSFER_FAILED', 'transactions', v_txn_id,
                'Credit to account ' || p_to_account_id || ' failed. Debit reversed.');

        COMMIT; -- Commit the FAILED status
        p_status  := 'FAILED';
        p_message := 'Credit to destination account failed. Transfer reversed.';
        RETURN;
    END IF;

    -- --------------------------------------------------------
    -- Step 9: Mark transaction as COMPLETED
    -- --------------------------------------------------------
    UPDATE transactions
    SET    status       = 'COMPLETED',
           completed_at = SYSTIMESTAMP
    WHERE  txn_id = v_txn_id;

    -- --------------------------------------------------------
    -- Step 10: Final audit log — transfer completed
    -- --------------------------------------------------------
    INSERT INTO audit_logs (event_type, table_name, record_id,
                             old_value, new_value, description)
    VALUES (
        'TRANSFER_COMPLETED', 'transactions', v_txn_id,
        'From Balance Before: ' || v_from_balance,
        'From Balance After: '  || (v_from_balance - v_total_debit),
        'Transfer of ' || p_amount || ' completed. Fee: ' || p_txn_fee
    );

    -- --------------------------------------------------------
    -- Step 11: COMMIT all changes
    -- --------------------------------------------------------
    COMMIT;

    p_status  := 'SUCCESS';
    p_message := 'Transfer of $' || p_amount
                 || ' completed. Reference: ' || p_txn_reference;

-- ============================================================
-- EXCEPTION HANDLERS
-- ============================================================
EXCEPTION
    WHEN ex_invalid_amount THEN
        ROLLBACK;
        p_status  := 'FAILED';
        p_message := 'Invalid transfer amount. Amount must be greater than zero.';

    WHEN ex_invalid_fee THEN
        ROLLBACK;
        p_status  := 'FAILED';
        p_message := 'Invalid fee amount. Fee cannot be negative.';

    WHEN ex_same_account THEN
        ROLLBACK;
        p_status  := 'FAILED';
        p_message := 'Source and destination accounts cannot be the same.';

    WHEN ex_account_not_found THEN
        ROLLBACK;
        p_status  := 'FAILED';
        p_message := 'One or both accounts were not found.';

    WHEN ex_account_frozen THEN
        ROLLBACK;
        p_status  := 'FAILED';
        p_message := 'Transfer rejected: one or both accounts are not ACTIVE.';

    WHEN ex_customer_suspended THEN
        ROLLBACK;
        p_status  := 'FAILED';
        p_message := 'Transfer rejected: source customer account is SUSPENDED.';

    WHEN ex_insufficient_funds THEN
        ROLLBACK;
        p_status  := 'FAILED';
        p_message := 'Insufficient funds. Balance after transfer would fall below minimum.';

    WHEN OTHERS THEN
        -- Catch-all for unexpected errors (ORA-00060 deadlock, ORA-08177, etc.)
        ROLLBACK;
        p_status  := 'ERROR';
        p_message := 'Unexpected error: SQLCODE=' || SQLCODE
                     || ', SQLERRM=' || SUBSTR(SQLERRM, 1, 200);

        -- Log the unexpected error
        BEGIN
            INSERT INTO audit_logs (event_type, description)
            VALUES ('SYSTEM_ERROR',
                    'Transfer failed unexpectedly. Ref: ' || NVL(p_txn_reference, 'N/A')
                    || ' Error: ' || SUBSTR(SQLERRM, 1, 500));
            COMMIT;
        EXCEPTION
            WHEN OTHERS THEN NULL; -- Don't let logging error mask original
        END;
END transfer_funds;
/
```

### 11.5 Execute the Transfer and Verify

```sql
-- ============================================================
-- TEST 1: SUCCESSFUL TRANSFER (Alice → Bob, $500)
-- ============================================================
DECLARE
    v_ref     VARCHAR2(100);
    v_status  VARCHAR2(20);
    v_message VARCHAR2(500);
BEGIN
    transfer_funds(
        p_from_account_id => 1,       -- Alice
        p_to_account_id   => 2,       -- Bob
        p_amount          => 500.00,
        p_description     => 'Rent payment',
        p_txn_fee         => 2.50,
        p_txn_reference   => v_ref,
        p_status          => v_status,
        p_message         => v_message
    );

    DBMS_OUTPUT.PUT_LINE('Status:    ' || v_status);
    DBMS_OUTPUT.PUT_LINE('Message:   ' || v_message);
    DBMS_OUTPUT.PUT_LINE('Reference: ' || v_ref);
END;
/
```

**Expected Output (Test 1):**

```
Status:    SUCCESS
Message:   Transfer of $500 completed. Reference: TXN-20260531143022123-1-2
Reference: TXN-20260531143022123-1-2
```

```sql
-- ============================================================
-- VERIFY BALANCES AFTER SUCCESSFUL TRANSFER
-- ============================================================
SELECT
    a.account_id,
    a.account_number,
    c.full_name,
    a.balance  AS current_balance
FROM accounts a
JOIN customers c ON c.customer_id = a.customer_id
ORDER BY a.account_id;
```

**Expected Output (After Test 1):**

| ACCOUNT_ID | ACCOUNT_NUMBER | FULL_NAME | CURRENT_BALANCE |
|---|---|---|---|
| 1 | ACC-001-ALICE | Alice Johnson | 4497.50 |
| 2 | ACC-002-BOB | Bob Martinez | 2500.00 |
| 3 | ACC-003-CAROL | Carol Williams | 500.00 |

> Alice: 5000.00 − 500.00 (transfer) − 2.50 (fee) = 4497.50  
> Bob: 2000.00 + 500.00 = 2500.00

```sql
-- ============================================================
-- VERIFY TRANSACTION LOG
-- ============================================================
SELECT
    txn_id,
    txn_reference,
    txn_type,
    from_account_id,
    to_account_id,
    amount,
    fee_amount,
    status,
    TO_CHAR(created_at, 'YYYY-MM-DD HH24:MI:SS') AS created_at
FROM transactions
ORDER BY txn_id;
```

**Expected Output:**

| TXN_ID | TXN_TYPE | FROM_ACC | TO_ACC | AMOUNT | FEE | STATUS | CREATED_AT |
|---|---|---|---|---|---|---|---|
| 1 | TRANSFER | 1 | 2 | 500.00 | 2.50 | COMPLETED | 2026-05-31 14:30:22 |

```sql
-- ============================================================
-- VERIFY AUDIT LOG
-- ============================================================
SELECT
    log_id,
    event_type,
    record_id,
    description,
    TO_CHAR(created_at, 'HH24:MI:SS') AS time
FROM audit_logs
ORDER BY log_id;
```

**Expected Output:**

| LOG_ID | EVENT_TYPE | RECORD_ID | DESCRIPTION | TIME |
|---|---|---|---|---|
| 1 | TRANSFER_INITIATED | 1 | Transfer from account 1 to 2 | 14:30:22 |
| 2 | TRANSFER_COMPLETED | 1 | Transfer of 500 completed. Fee: 2.5 | 14:30:22 |

```sql
-- ============================================================
-- TEST 2: FAILED TRANSFER — INSUFFICIENT FUNDS
-- ============================================================
DECLARE
    v_ref     VARCHAR2(100);
    v_status  VARCHAR2(20);
    v_message VARCHAR2(500);
BEGIN
    transfer_funds(
        p_from_account_id => 2,       -- Bob (balance: 2500)
        p_to_account_id   => 1,       -- Alice
        p_amount          => 10000.00, -- Way more than Bob has!
        p_description     => 'Large transfer attempt',
        p_txn_fee         => 2.50,
        p_txn_reference   => v_ref,
        p_status          => v_status,
        p_message         => v_message
    );

    DBMS_OUTPUT.PUT_LINE('Status:  ' || v_status);
    DBMS_OUTPUT.PUT_LINE('Message: ' || v_message);
END;
/
```

**Expected Output (Test 2):**

```
Status:  FAILED
Message: Insufficient funds. Balance after transfer would fall below minimum.
```

Balances remain **unchanged** — ROLLBACK was triggered.

```sql
-- ============================================================
-- TEST 3: FAILED TRANSFER — SUSPENDED CUSTOMER
-- ============================================================
DECLARE
    v_ref     VARCHAR2(100);
    v_status  VARCHAR2(20);
    v_message VARCHAR2(500);
BEGIN
    transfer_funds(
        p_from_account_id => 3,       -- Carol (SUSPENDED customer)
        p_to_account_id   => 1,
        p_amount          => 100.00,
        p_txn_reference   => v_ref,
        p_status          => v_status,
        p_message         => v_message
    );

    DBMS_OUTPUT.PUT_LINE('Status:  ' || v_status);
    DBMS_OUTPUT.PUT_LINE('Message: ' || v_message);
END;
/
```

**Expected Output (Test 3):**

```
Status:  FAILED
Message: Transfer rejected: source customer account is SUSPENDED.
```

### 11.6 Step-by-Step Transaction Flow Diagram

```
transfer_funds() called
        │
        ▼
[1] Generate TXN Reference
        │
        ▼
[2] Validate Input (amount > 0, different accounts)
        │── FAIL ──► ROLLBACK → Return error
        ▼
SAVEPOINT sp_before_transfer
        │
        ▼
[3] SELECT ... FOR UPDATE (lock both accounts, ordered by ID)
        │── NOT FOUND ──► ROLLBACK → Return error
        ▼
[4] Business Rule Validation (status, customer, balance check)
        │── FAIL ──► ROLLBACK → Return error
        ▼
[5] INSERT INTO transactions (status = PENDING)
        │
        ▼
SAVEPOINT sp_txn_recorded
        │
        ▼
[6] INSERT INTO audit_logs (TRANSFER_INITIATED)
        │
        ▼
SAVEPOINT sp_audit_initiated
        │
        ▼
[7] UPDATE accounts — DEBIT source account
        │── 0 ROWS ──► ROLLBACK TO sp_before_transfer → Return error
        ▼
SAVEPOINT sp_debited
        │
        ▼
[8] UPDATE accounts — CREDIT destination account
        │── 0 ROWS ──► ROLLBACK TO sp_audit_initiated
        │               UPDATE txn status = FAILED
        │               INSERT audit (TRANSFER_FAILED)
        │               COMMIT (failed status) → Return FAILED
        ▼
[9] UPDATE transactions — status = COMPLETED
        │
        ▼
[10] INSERT INTO audit_logs (TRANSFER_COMPLETED)
        │
        ▼
[11] COMMIT — All changes permanent
        │
        ▼
Return SUCCESS

EXCEPTION block:
  Any unexpected error → ROLLBACK → Log to audit → Return ERROR
```

---

## 12. Laravel Developer Section

### Why Laravel Developers Must Understand Oracle Transactions

Laravel's Eloquent ORM and query builder abstract database interactions beautifully — but they were designed primarily with MySQL and PostgreSQL in mind. When you work with Oracle in production (common in enterprise, banking, and government systems), several critical behavioral differences can cause silent data corruption, failed deployments, or unreliable rollbacks if you don't understand the Oracle transaction model.

Key differences:
- Oracle does **not** auto-commit by default in raw SQL sessions — but some Oracle drivers for PHP/Laravel **do** enable auto-commit by default.
- DDL in Oracle causes **implicit COMMIT** — mixing DDL with DML in a Laravel migration or seeder can silently commit partial work.
- Oracle's `SEQUENCES` behave differently from MySQL `AUTO_INCREMENT`.
- Laravel's `DB::transaction()` maps correctly to Oracle `COMMIT`/`ROLLBACK` — but only if your Oracle driver is configured to disable auto-commit.

### 12.1 Laravel `DB::transaction()`

This is the safest way to manage Oracle transactions in Laravel. If any exception is thrown inside the closure, Laravel automatically issues a `ROLLBACK`.

```php
use Illuminate\Support\Facades\DB;
use Exception;

/**
 * Transfer funds between two accounts using Laravel's transaction wrapper.
 *
 * @param int   $fromAccountId
 * @param int   $toAccountId
 * @param float $amount
 * @param float $fee
 * @throws Exception
 */
function transferFunds(int $fromAccountId, int $toAccountId, float $amount, float $fee = 2.50): array
{
    return DB::transaction(function () use ($fromAccountId, $toAccountId, $amount, $fee) {

        $totalDebit = $amount + $fee;

        // --------------------------------------------------
        // Step 1: Lock source account row (SELECT FOR UPDATE)
        // Oracle: SELECT ... FOR UPDATE WAIT 10
        // --------------------------------------------------
        $fromAccount = DB::selectOne(
            'SELECT account_id, balance, status, minimum_balance
             FROM accounts
             WHERE account_id = :id
             FOR UPDATE WAIT 10',
            ['id' => $fromAccountId]
        );

        if (!$fromAccount) {
            throw new Exception('Source account not found.');
        }

        // --------------------------------------------------
        // Step 2: Lock destination account row
        // --------------------------------------------------
        $toAccount = DB::selectOne(
            'SELECT account_id, balance, status
             FROM accounts
             WHERE account_id = :id
             FOR UPDATE WAIT 10',
            ['id' => $toAccountId]
        );

        if (!$toAccount) {
            throw new Exception('Destination account not found.');
        }

        // --------------------------------------------------
        // Step 3: Business rule validations
        // --------------------------------------------------
        if ($fromAccount->status !== 'ACTIVE') {
            throw new Exception('Source account is not active.');
        }

        if ($toAccount->status !== 'ACTIVE') {
            throw new Exception('Destination account is not active.');
        }

        $balanceAfterDebit = $fromAccount->balance - $totalDebit;
        if ($balanceAfterDebit < $fromAccount->minimum_balance) {
            throw new Exception(
                'Insufficient funds. Available: $' . ($fromAccount->balance - $fromAccount->minimum_balance)
            );
        }

        // --------------------------------------------------
        // Step 4: Generate transaction reference
        // --------------------------------------------------
        $txnReference = 'TXN-' . date('YmdHisu') . '-' . $fromAccountId . '-' . $toAccountId;

        // --------------------------------------------------
        // Step 5: Insert transaction record (PENDING)
        // --------------------------------------------------
        DB::insert(
            'INSERT INTO transactions
             (txn_reference, txn_type, from_account_id, to_account_id,
              amount, fee_amount, currency, status, description)
             VALUES (:ref, :type, :from, :to, :amount, :fee, :cur, :status, :desc)',
            [
                'ref'    => $txnReference,
                'type'   => 'TRANSFER',
                'from'   => $fromAccountId,
                'to'     => $toAccountId,
                'amount' => $amount,
                'fee'    => $fee,
                'cur'    => 'USD',
                'status' => 'PENDING',
                'desc'   => 'Fund Transfer via Laravel',
            ]
        );

        // --------------------------------------------------
        // Step 6: Debit source account
        // --------------------------------------------------
        DB::update(
            'UPDATE accounts SET balance = balance - :debit WHERE account_id = :id',
            ['debit' => $totalDebit, 'id' => $fromAccountId]
        );

        // --------------------------------------------------
        // Step 7: Credit destination account
        // --------------------------------------------------
        DB::update(
            'UPDATE accounts SET balance = balance + :credit WHERE account_id = :id',
            ['credit' => $amount, 'id' => $toAccountId]
        );

        // --------------------------------------------------
        // Step 8: Mark transaction as COMPLETED
        // --------------------------------------------------
        DB::update(
            "UPDATE transactions
             SET status = 'COMPLETED', completed_at = SYSTIMESTAMP
             WHERE txn_reference = :ref",
            ['ref' => $txnReference]
        );

        // --------------------------------------------------
        // Step 9: Audit log
        // --------------------------------------------------
        DB::insert(
            'INSERT INTO audit_logs (event_type, table_name, description)
             VALUES (:event, :table, :desc)',
            [
                'event' => 'TRANSFER_COMPLETED',
                'table' => 'transactions',
                'desc'  => 'Transfer of $' . $amount . ' completed. Ref: ' . $txnReference,
            ]
        );

        // DB::transaction() will COMMIT here if no exception was thrown
        // If any step above threw an Exception, Laravel calls ROLLBACK automatically

        return [
            'status'    => 'SUCCESS',
            'reference' => $txnReference,
            'message'   => 'Transfer of $' . $amount . ' completed successfully.',
        ];

    }); // End DB::transaction()
}

// ----------------------------------------------------------
// Usage in a Controller
// ----------------------------------------------------------
try {
    $result = transferFunds(fromAccountId: 1, toAccountId: 2, amount: 500.00);
    return response()->json($result, 200);
} catch (Exception $e) {
    return response()->json([
        'status'  => 'FAILED',
        'message' => $e->getMessage(),
    ], 422);
}
```

### 12.2 Manual Transaction Control in Laravel

For complex flows where you need finer control (similar to Oracle SAVEPOINTs):

```php
use Illuminate\Support\Facades\DB;

DB::beginTransaction(); // Oracle: transaction starts implicitly, this sets the savepoint

try {
    // --- Step 1: Insert order header ---
    DB::insert(
        'INSERT INTO orders (order_id, customer_id, status)
         VALUES (orders_seq.NEXTVAL, :cid, :status)',
        ['cid' => 55, 'status' => 'PENDING']
    );

    // --- Step 2: Insert order items ---
    foreach ($cartItems as $item) {
        DB::insert(
            'INSERT INTO order_items (order_id, product_id, qty, price)
             VALUES (orders_seq.CURRVAL, :pid, :qty, :price)',
            ['pid' => $item['product_id'], 'qty' => $item['qty'], 'price' => $item['price']]
        );
    }

    // --- Step 3: Deduct stock ---
    foreach ($cartItems as $item) {
        $affected = DB::update(
            'UPDATE products
             SET stock = stock - :qty
             WHERE product_id = :pid AND stock >= :qty',
            ['qty' => $item['qty'], 'pid' => $item['product_id']]
        );

        if ($affected === 0) {
            throw new Exception('Insufficient stock for product ' . $item['product_id']);
        }
    }

    DB::commit(); // Maps to Oracle COMMIT

    return ['status' => 'Order placed successfully.'];

} catch (Exception $e) {
    DB::rollBack(); // Maps to Oracle ROLLBACK

    return ['status' => 'FAILED', 'message' => $e->getMessage()];
}
```

### 12.3 Calling an Oracle Stored Procedure from Laravel

```php
use Illuminate\Support\Facades\DB;

/**
 * Call the Oracle transfer_funds stored procedure from Laravel.
 */
function callOracleTransferProcedure(
    int   $fromAccountId,
    int   $toAccountId,
    float $amount,
    float $fee = 2.50
): array {
    // Bind output parameters using OCI8 or PDO with named parameters
    $txnReference = '';
    $status       = '';
    $message      = '';

    // Using yajra/laravel-oci8 package (most common Oracle driver for Laravel)
    $result = DB::executeProcedure('transfer_funds', [
        'p_from_account_id' => $fromAccountId,
        'p_to_account_id'   => $toAccountId,
        'p_amount'          => $amount,
        'p_description'     => 'Transfer via Laravel',
        'p_txn_fee'         => $fee,
        'p_txn_reference'   => &$txnReference,  // OUT parameter
        'p_status'          => &$status,         // OUT parameter
        'p_message'         => &$message,        // OUT parameter
    ]);

    return [
        'reference' => $txnReference,
        'status'    => $status,
        'message'   => $message,
    ];
}

// Alternative: Raw OCI8 call
function callProcedureRaw(int $fromId, int $toId, float $amount): array
{
    $pdo = DB::getPdo(); // Returns the underlying PDO connection

    $stmt = $pdo->prepare(
        'BEGIN transfer_funds(
            :from, :to, :amount, :desc, :fee,
            :ref, :status, :message
         ); END;'
    );

    $txnRef  = '';
    $status  = '';
    $message = '';

    $stmt->bindParam(':from',    $fromId,   PDO::PARAM_INT);
    $stmt->bindParam(':to',      $toId,     PDO::PARAM_INT);
    $stmt->bindParam(':amount',  $amount,   PDO::PARAM_STR);
    $stmt->bindParam(':desc',    $desc = 'Transfer', PDO::PARAM_STR);
    $stmt->bindParam(':fee',     $fee = 2.5, PDO::PARAM_STR);
    $stmt->bindParam(':ref',     $txnRef,   PDO::PARAM_STR|PDO::PARAM_INPUT_OUTPUT, 100);
    $stmt->bindParam(':status',  $status,   PDO::PARAM_STR|PDO::PARAM_INPUT_OUTPUT, 20);
    $stmt->bindParam(':message', $message,  PDO::PARAM_STR|PDO::PARAM_INPUT_OUTPUT, 500);

    $stmt->execute();

    return compact('txnRef', 'status', 'message');
}
```

### 12.4 How Laravel Transaction Handling Maps to Oracle

| Laravel Method | Oracle Equivalent | Notes |
|---|---|---|
| `DB::beginTransaction()` | No-op (transaction starts on first DML) | Marks start for Laravel's tracking |
| `DB::commit()` | `COMMIT` | Makes all changes permanent |
| `DB::rollBack()` | `ROLLBACK` | Undoes all changes since last commit |
| `DB::transaction(fn)` | `COMMIT` on success / `ROLLBACK` on exception | Safest approach |
| No Laravel wrapper | Oracle requires explicit `COMMIT` or `ROLLBACK` | Easy to forget! |

### 12.5 Common Mistakes Laravel Developers Make with Oracle

#### Mistake 1: Forgetting That Oracle Requires Explicit COMMIT

```php
// ❌ WRONG — In Oracle, this UPDATE is NOT auto-committed!
// The connection closes, Oracle auto-rollbacks the changes silently.
DB::statement('UPDATE accounts SET balance = 9999 WHERE account_id = 1');
// No commit → on connection close → ROLLBACK

// ✅ CORRECT
DB::statement('UPDATE accounts SET balance = 9999 WHERE account_id = 1');
DB::statement('COMMIT');
// Or use DB::transaction() which handles this automatically
```

#### Mistake 2: Mixing DDL Inside a Laravel DB::transaction() Block

```php
// ❌ WRONG — Oracle DDL causes implicit COMMIT!
DB::transaction(function () {
    DB::insert('INSERT INTO orders ...'); // Step 1
    DB::statement('CREATE TABLE temp_work ...'); // Step 2 — COMMITS Step 1 immediately!
    throw new Exception('Something failed');
    // DB::rollBack() called here — but Step 1 is ALREADY committed!
});

// ✅ CORRECT — Keep DDL completely outside transactions
DB::statement('CREATE TABLE temp_work (id NUMBER)'); // DDL outside transaction
DB::transaction(function () {
    DB::insert('INSERT INTO orders ...'); // Safe in transaction
});
```

#### Mistake 3: Not Using Row Locks When Updating Balances

```php
// ❌ WRONG — Race condition: two requests can read the same balance
$account = DB::selectOne('SELECT balance FROM accounts WHERE account_id = :id', ['id' => 1]);
DB::update('UPDATE accounts SET balance = :bal WHERE account_id = :id', [
    'bal' => $account->balance - 500,
    'id'  => 1
]);
// Two concurrent requests both read balance=5000, both subtract 500, final balance is 4500 not 4000

// ✅ CORRECT — Lock the row before reading
DB::transaction(function () {
    $account = DB::selectOne(
        'SELECT balance FROM accounts WHERE account_id = :id FOR UPDATE WAIT 10',
        ['id' => 1]
    );
    DB::update('UPDATE accounts SET balance = balance - 500 WHERE account_id = :id', ['id' => 1]);
});
```

#### Mistake 4: Assuming MySQL `TRUNCATE` Behavior in Oracle Rollback

```php
// ❌ WRONG assumption — TRUNCATE cannot be rolled back in Oracle
DB::transaction(function () {
    DB::statement('TRUNCATE TABLE temp_staging'); // Implicit COMMIT — cannot roll back!
    DB::insert('INSERT INTO temp_staging ...');
    // If this insert fails and rollback is called, the TRUNCATE is ALREADY permanent
});

// ✅ CORRECT — Use DELETE inside transactions if you need rollback capability
DB::transaction(function () {
    DB::delete('DELETE FROM temp_staging'); // DML — can be rolled back
    DB::insert('INSERT INTO temp_staging ...');
});
```

#### Mistake 5: Long-Running Transactions

```php
// ❌ WRONG — Holding locks for the duration of a slow loop
DB::beginTransaction();
foreach ($millionRecords as $record) {
    DB::update('UPDATE accounts SET balance = ...', [...]); // Row locked throughout
    // Thousands of rows locked for minutes → blocks other sessions!
}
DB::commit();

// ✅ CORRECT — Batch commits to release locks regularly
$batchSize = 1000;
$batch = [];
foreach ($millionRecords as $i => $record) {
    $batch[] = $record;
    if (count($batch) >= $batchSize) {
        DB::transaction(function () use ($batch) {
            foreach ($batch as $r) {
                DB::update('UPDATE accounts SET balance = ...', [...]);
            }
        });
        $batch = [];
    }
}
```

---

## 13. Practical Real-Life Use Cases

### Banking Fund Transfers
Every wire transfer, ACH payment, or internal account move requires atomicity. A debit without a credit, or vice versa, is a financial disaster. Oracle transactions guarantee both legs succeed or neither does.

### E-Commerce Checkout
The order placement pipeline typically involves: creating the order record → reserving inventory → processing payment → sending confirmation. Oracle transactions wrap all of these operations so that a payment failure automatically restores stock and cancels the order record.

### Inventory Stock Deduction
When multiple concurrent customers order the same limited-stock item, `SELECT ... FOR UPDATE` prevents overselling by ensuring only one session can deduct stock at a time. The check and deduction are atomic.

### Payroll Processing
Monthly payroll must credit every employee exactly once. A transaction wraps the payroll run, and if any employee's salary computation or bank routing fails validation, the entire batch can be rolled back and corrected before committing.

### Student Admission and Payment Confirmation
University admission systems create an admission record, deduct a seat from available quota, record payment, and generate a confirmation — all in one transaction. A failed payment reverses the seat reservation automatically.

### Hospital Billing
When a patient is discharged, the billing system simultaneously closes the patient encounter, generates itemized charges, updates insurance claims, and posts the bill — all or nothing. Partial billing creates compliance and audit risks.

### Loan Approval Workflow
Loan approval involves credit scoring, document verification, approval by multiple officers, and finally disbursement. Each stage transition is a transaction. If the disbursement fails, the approval status is rolled back using `SAVEPOINT`-style control at the application level.

### Approval-Based Enterprise Systems
Enterprise resource planning (ERP) purchase orders, budget approvals, and journal entries use transactions to ensure that a state change (e.g., DRAFT → APPROVED → POSTED) is consistent across the workflow table, the ledger, and the audit log simultaneously.

---

## 14. Best Practices

| Practice | Why It Matters |
|---|---|
| Always use `SELECT ... FOR UPDATE` before updating balance/stock | Prevents lost updates and race conditions |
| Lock multiple rows in a consistent order (by PK) | Prevents deadlocks |
| Use `FOR UPDATE WAIT n` instead of `NOWAIT` | Avoids aggressive failures under brief contention |
| Keep transactions short and focused | Minimizes lock contention and undo segment usage |
| Always handle `WHEN OTHERS` in PL/SQL | Ensures unexpected errors trigger ROLLBACK, not silent half-commits |
| Never mix DDL and DML in the same transaction | DDL causes implicit COMMIT of preceding DML |
| Use `SAVEPOINT` for multi-stage workflows | Allows partial rollback without losing all work |
| Use named transactions (`SET TRANSACTION NAME`) | Makes tracing and debugging easier in production |
| Always write to an audit log inside the transaction | Audit entries are committed or rolled back with the main data |
| Increase `UNDO_RETENTION` for long reports | Prevents `ORA-01555: snapshot too old` |
| Catch and retry on `ORA-00060` (deadlock) | Oracle only rolls back one statement — you must retry explicitly |
| Test with concurrent sessions, not just sequential | Race conditions only appear under load |

---

## 15. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Forgetting `COMMIT` after DML in SQL*Plus | Changes lost on disconnect | Always explicitly COMMIT or use `SET AUTOCOMMIT ON` for scripts |
| Running DDL inside a DML transaction expecting rollback | DDL auto-commits DML silently | Keep DDL in separate scripts outside transactions |
| Using `TRUNCATE` expecting rollback | Data permanently deleted | Use `DELETE` inside transactions; use `TRUNCATE` only when rollback is not needed |
| Not using `FOR UPDATE` on balance updates | Lost updates under concurrency | Always lock before read-then-update patterns |
| Assuming `ROLLBACK` undoes DDL | It does not in Oracle | Test schema changes in a dev environment first |
| Ignoring `ORA-01555` in long transactions | Query fails mid-report | Increase undo retention; refactor long queries |
| Using auto-commit in production OLTP code | Cannot undo multi-step failures | Disable auto-commit; use explicit transaction management |
| Holding transactions open across HTTP requests | Long-running locks block other users | Open transaction, do work, commit/rollback — all in one request |
| Not logging errors from `WHEN OTHERS` | Silent failures, impossible to debug | Always log `SQLCODE` and `SQLERRM` to an audit/error table |
| Assuming Oracle and MySQL behave identically | Subtle data integrity bugs | Test explicitly against Oracle; don't port MySQL code blindly |

---

## 16. Conclusion

Oracle Transaction Management is one of the most powerful and nuanced areas of Oracle Database. Its MVCC architecture, row-level locking, automatic deadlock detection, and multi-granularity savepoints make it exceptionally capable for high-concurrency, mission-critical applications.

**Key takeaways:**

- A transaction is **atomic** — all steps succeed or all are rolled back.
- Oracle does **not auto-commit DML** — you must explicitly `COMMIT`.
- DDL (`CREATE`, `ALTER`, `DROP`, `TRUNCATE`) **always triggers an implicit COMMIT** — never mix with rollback-critical DML.
- Use `SELECT ... FOR UPDATE` to **lock rows before modifying** them — this is the foundation of safe concurrent updates.
- Use **`SAVEPOINT`** for complex multi-stage workflows where partial rollback is needed.
- Use **`SERIALIZABLE`** isolation for financial consistency reports; use `READ COMMITTED` (the default) for OLTP.
- Handle `WHEN OTHERS` in every PL/SQL block — log `SQLCODE` and `SQLERRM`, then `ROLLBACK`.
- In Laravel, `DB::transaction()` correctly maps to Oracle's `COMMIT`/`ROLLBACK`, but you must understand the Oracle-specific behaviors that Laravel's abstraction does not protect you from.

Mastering Oracle transactions is not optional for anyone building production systems on Oracle. It is the difference between a system that handles failures gracefully and one that silently corrupts data under load.

---

*Document prepared for advanced Oracle and Laravel developers. All SQL examples use Oracle Database syntax and are tested against Oracle Database 19c and 21c.*
