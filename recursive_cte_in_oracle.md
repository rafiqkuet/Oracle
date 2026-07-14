# Oracle Recursive CTE (Recursive Subquery Factoring) — Advanced Guide

> A complete, practical, and production-ready reference for Oracle developers and Laravel engineers working with hierarchical data.

---

## Table of Contents

1. [Introduction](#introduction)
2. [What Is a Recursive CTE?](#what-is-a-recursive-cte)
3. [Why Oracle Calls It Recursive Subquery Factoring](#why-oracle-calls-it-recursive-subquery-factoring)
4. [The Role of the WITH Clause](#the-role-of-the-with-clause)
5. [Anchor Query and Recursive Query](#anchor-query-and-recursive-query)
6. [How Recursion Stops](#how-recursion-stops)
7. [Key Differences Explained](#key-differences-explained)
8. [When to Use Recursive CTE Instead of CONNECT BY](#when-to-use-recursive-cte-instead-of-connect-by)
9. [Oracle Recursive CTE Syntax — Step by Step](#oracle-recursive-cte-syntax--step-by-step)
10. [Full Table Design — Enterprise Employee Hierarchy](#full-table-design--enterprise-employee-hierarchy)
11. [Sample Data](#sample-data)
12. [Recursive CTE Examples](#recursive-cte-examples)
    - [Example 1: Complete Organization Tree](#example-1-complete-organization-tree)
    - [Example 2: All Subordinates Under a Manager](#example-2-all-subordinates-under-a-manager)
    - [Example 3: Approval Chain Upward to CEO](#example-3-approval-chain-upward-to-ceo)
    - [Example 4: Salary Roll-Up Per Manager](#example-4-salary-roll-up-per-manager)
    - [Example 5: Leaf Nodes — Employees Without Subordinates](#example-5-leaf-nodes--employees-without-subordinates)
    - [Example 6: Distance Between Two Employees](#example-6-distance-between-two-employees)
    - [Example 7: Cycle Detection](#example-7-cycle-detection)
13. [Product Category Tree Example](#product-category-tree-example)
14. [Recursive CTE vs CONNECT BY — Detailed Comparison](#recursive-cte-vs-connect-by--detailed-comparison)
15. [Performance Tuning](#performance-tuning)
16. [Laravel Developer Integration](#laravel-developer-integration)
17. [Practical Real-Life Use Cases](#practical-real-life-use-cases)
18. [Comparison Tables](#comparison-tables)
19. [Common Mistakes and Fixes](#common-mistakes-and-fixes)
20. [Best Practices](#best-practices)
21. [Conclusion](#conclusion)

---

## Introduction

Hierarchical and recursive data is everywhere in enterprise systems: employee reporting structures, product category trees, approval workflows, referral networks, bill of materials, menu systems, and comment threads. Oracle Database has supported hierarchical queries since very early versions using the proprietary `CONNECT BY` clause. Starting with **Oracle 11g Release 2**, Oracle introduced **Recursive Subquery Factoring** — the SQL-standard compliant way to handle recursive queries using the `WITH` clause.

This guide covers Recursive CTE deeply, from syntax fundamentals to a fully working enterprise example, performance tuning, and practical Laravel integration.

---

## What Is a Recursive CTE?

A **Common Table Expression (CTE)** is a named temporary result set that you define inside a query using the `WITH` clause. A **Recursive CTE** is a CTE that references itself — it keeps executing a query repeatedly, each time using the result from the previous iteration as input, until a stopping condition is met.

In plain terms:

- It starts with a **base result** (the anchor query).
- It then repeatedly **applies a recursive step** (the recursive query) to expand the result set.
- It **stops** when the recursive step produces no new rows.

This is ideal for data that has a **parent-child relationship** across an unknown or variable number of levels.

---

## Why Oracle Calls It Recursive Subquery Factoring

Oracle's documentation uses the term **Recursive Subquery Factoring** rather than "Recursive CTE". This is because:

- **Subquery Factoring** is Oracle's official name for the `WITH` clause feature — it allows you to *factor out* (define separately) a subquery that might otherwise appear multiple times in your main query.
- When that factored subquery references itself, it becomes **recursive subquery factoring**.

The term "CTE" (Common Table Expression) is the ANSI SQL standard terminology. Oracle supports the same concept but names it differently in its official documentation. Both terms refer to the same feature.

> **In practice**, developers use "Recursive CTE" and "Recursive Subquery Factoring" interchangeably when working with Oracle.

---

## The Role of the WITH Clause

The `WITH` clause allows you to define one or more **named query blocks** before the main `SELECT`. These named blocks are:

- Computed once (or iteratively, in the recursive case).
- Referenceable by name in the main query and in subsequent `WITH` blocks.
- Scoped to the current SQL statement — they do not persist.

```sql
WITH
  dept_summary AS (
    SELECT department_id, COUNT(*) AS emp_count
    FROM employees
    GROUP BY department_id
  ),
  high_count_depts AS (
    SELECT department_id
    FROM dept_summary
    WHERE emp_count > 10
  )
SELECT *
FROM high_count_depts;
```

In the recursive case, the `WITH` clause defines a CTE that references itself, enabling iterative traversal of hierarchical data.

---

## Anchor Query and Recursive Query

Every Recursive CTE has exactly **two parts**, combined with `UNION ALL`:

### 1. Anchor Query (Base Case)

The anchor query is a plain `SELECT` that produces the **starting rows** of the recursion. It does **not** reference the CTE name. It establishes the first level of the hierarchy.

```sql
-- Anchor: select the root node (CEO has no manager)
SELECT employee_id, employee_name, manager_id, 1 AS lvl
FROM employees
WHERE manager_id IS NULL
```

### 2. Recursive Query

The recursive query **joins the CTE name back to the source table**, using the results from the previous iteration as the "parent" side. Each iteration produces the next level of children.

```sql
-- Recursive: for each row already found, find its children
SELECT e.employee_id, e.employee_name, e.manager_id, r.lvl + 1
FROM employees e
JOIN emp_hierarchy r ON e.manager_id = r.employee_id
```

These two parts are always connected with `UNION ALL`.

---

## How Recursion Stops

Oracle's recursive CTE execution works like this:

1. Execute the **anchor query** → produces an initial result set (Iteration 0).
2. Use Iteration 0 as input to the **recursive query** → produces Iteration 1.
3. Use Iteration 1 as input → produces Iteration 2.
4. Continue until the recursive query **returns zero rows**.
5. Combine all iterations using `UNION ALL` → final result.

The recursion stops naturally when no more child rows are found. If your data has circular references, Oracle will loop indefinitely (or until memory is exhausted) unless you add **cycle detection** or a **depth limit**.

---

## Key Differences Explained

### Normal CTE vs Recursive CTE

| Aspect | Normal CTE | Recursive CTE |
|---|---|---|
| Self-reference | No | Yes |
| `SEARCH` / `CYCLE` clause | Not applicable | Optional, for ordering and cycle detection |
| Use case | Simplifying complex queries | Hierarchical / recursive data traversal |
| Execution | Once | Iteratively until empty result |
| `UNION ALL` required | No | Yes (between anchor and recursive parts) |

### Recursive CTE vs Hierarchical Query

Both traverse hierarchical data. "Hierarchical query" is a broad term — in Oracle specifically, it usually refers to `SELECT ... START WITH ... CONNECT BY`. Recursive CTE is the SQL-standard alternative that is more flexible and portable.

### Recursive CTE vs CONNECT BY

| Feature | Recursive CTE | CONNECT BY |
|---|---|---|
| SQL Standard | ANSI SQL:1999 compliant | Oracle-proprietary |
| Portability | Works on PostgreSQL, SQL Server, DB2, MySQL 8+ | Oracle only |
| Complex JOINs in recursion | Fully supported | Limited |
| Aggregation during traversal | Supported | Not directly |
| Cycle detection | Manual (counter or `CYCLE` clause) | `NOCYCLE` keyword |
| Path generation | String concatenation | `SYS_CONNECT_BY_PATH()` |
| Multiple tables in recursion | Easy | Complex / workarounds needed |
| Oracle-specific functions | Works alongside | `LEVEL`, `PRIOR`, `CONNECT_BY_ROOT` |

### UNION ALL vs UNION in Recursive CTE

You **must always use `UNION ALL`** in a Recursive CTE. Here is why:

- `UNION ALL` keeps all rows from each iteration, including duplicates. This is required for the recursive engine to process all rows.
- `UNION` would attempt to eliminate duplicates, which forces Oracle to materialize and sort the entire result set. Oracle **does not allow `UNION` in the recursive part** of a CTE — it raises an error (`ORA-32039: recursive WITH clause must use a UNION ALL`).

---

## When to Use Recursive CTE Instead of CONNECT BY

Use **Recursive CTE** when:

- You need **portability** — your code may run on multiple database platforms.
- The recursive step involves **complex JOINs** across multiple tables.
- You need **aggregation** or **computed columns** that depend on parent rows.
- You are performing **bottom-up traversal** (child to root).
- You need to **pass computed values down** through hierarchy levels.
- Your team uses modern SQL and you prefer the standard approach.

Use **CONNECT BY** when:

- You are writing **Oracle-only** code and want brevity.
- You need quick access to Oracle-specific pseudo-columns like `LEVEL`, `CONNECT_BY_ROOT`, `SYS_CONNECT_BY_PATH`.
- The query is simple and the hierarchy is in a single table.
- You are maintaining **legacy Oracle code**.

---

## Oracle Recursive CTE Syntax — Step by Step

```sql
WITH recursive_cte_name (column1, column2, level_no) AS (
    -- Anchor query
    SELECT column1, column2, 1 AS level_no
    FROM table_name
    WHERE parent_id IS NULL

    UNION ALL

    -- Recursive query
    SELECT child.column1, child.column2, parent.level_no + 1
    FROM table_name child
    JOIN recursive_cte_name parent
        ON child.parent_id = parent.column1
)
SELECT *
FROM recursive_cte_name;
```

### Line-by-Line Explanation

| Part | Explanation |
|---|---|
| `WITH recursive_cte_name` | Declares the CTE name. This name is used in the recursive query to reference itself. |
| `(column1, column2, level_no)` | Explicit column list — required in Oracle when the CTE is recursive. Column names must match the number and order of columns returned by the SELECT. |
| Anchor `SELECT` | Selects the starting rows. `WHERE parent_id IS NULL` finds root nodes (top of hierarchy). `1 AS level_no` seeds the level counter. |
| `UNION ALL` | Connects anchor and recursive parts. Required — `UNION` is not allowed. |
| Recursive `SELECT` | Joins the source table (`child`) to the CTE itself (`parent`). `parent.level_no + 1` increments the depth counter each iteration. |
| `JOIN recursive_cte_name parent` | This is the self-reference that makes it recursive. Oracle uses the previous iteration's rows as the `parent` side. |
| `ON child.parent_id = parent.column1` | The parent-child join condition. Each iteration finds children of the previously found rows. |
| Final `SELECT * FROM recursive_cte_name` | Reads the accumulated result of all iterations. |

---

## Full Table Design — Enterprise Employee Hierarchy

```sql
-- ============================================================
-- TABLE: DEPARTMENTS
-- ============================================================
CREATE TABLE departments (
    department_id   NUMBER(10)      NOT NULL,
    department_name VARCHAR2(100)   NOT NULL,
    location        VARCHAR2(100),
    CONSTRAINT pk_departments PRIMARY KEY (department_id),
    CONSTRAINT uq_dept_name UNIQUE (department_name)
);

-- ============================================================
-- TABLE: EMPLOYEES
-- ============================================================
CREATE TABLE employees (
    employee_id     NUMBER(10)      NOT NULL,
    employee_name   VARCHAR2(100)   NOT NULL,
    email           VARCHAR2(150)   NOT NULL,
    job_title       VARCHAR2(100)   NOT NULL,
    department_id   NUMBER(10)      NOT NULL,
    manager_id      NUMBER(10),                         -- NULL for CEO
    hire_date       DATE            NOT NULL,
    status          VARCHAR2(20)    DEFAULT 'ACTIVE'    NOT NULL,
    CONSTRAINT pk_employees       PRIMARY KEY (employee_id),
    CONSTRAINT uq_employee_email  UNIQUE (email),
    CONSTRAINT fk_emp_department  FOREIGN KEY (department_id)
        REFERENCES departments(department_id),
    CONSTRAINT fk_emp_manager     FOREIGN KEY (manager_id)
        REFERENCES employees(employee_id),
    CONSTRAINT chk_emp_status     CHECK (status IN ('ACTIVE','INACTIVE','ON_LEAVE'))
);

-- ============================================================
-- TABLE: EMPLOYEE_SALARY_HISTORY
-- ============================================================
CREATE TABLE employee_salary_history (
    salary_id       NUMBER(10)      NOT NULL,
    employee_id     NUMBER(10)      NOT NULL,
    salary_amount   NUMBER(12, 2)   NOT NULL,
    currency        VARCHAR2(10)    DEFAULT 'USD'       NOT NULL,
    effective_from  DATE            NOT NULL,
    effective_to    DATE,                               -- NULL means current
    CONSTRAINT pk_salary_history     PRIMARY KEY (salary_id),
    CONSTRAINT fk_salary_employee    FOREIGN KEY (employee_id)
        REFERENCES employees(employee_id),
    CONSTRAINT chk_salary_amount     CHECK (salary_amount > 0),
    CONSTRAINT chk_salary_currency   CHECK (currency IN ('USD','EUR','GBP','BDT'))
);

-- ============================================================
-- TABLE: APPROVAL_LIMITS
-- ============================================================
CREATE TABLE approval_limits (
    limit_id        NUMBER(10)      NOT NULL,
    employee_id     NUMBER(10)      NOT NULL,
    limit_type      VARCHAR2(50)    NOT NULL,           -- e.g. 'EXPENSE', 'HIRING', 'CONTRACT'
    max_amount      NUMBER(12, 2)   NOT NULL,
    CONSTRAINT pk_approval_limits   PRIMARY KEY (limit_id),
    CONSTRAINT fk_approval_employee FOREIGN KEY (employee_id)
        REFERENCES employees(employee_id),
    CONSTRAINT chk_limit_type       CHECK (limit_type IN ('EXPENSE','HIRING','CONTRACT','BUDGET'))
);

-- ============================================================
-- TABLE: AUDIT_LOGS
-- ============================================================
CREATE TABLE audit_logs (
    log_id          NUMBER(10)      NOT NULL,
    table_name      VARCHAR2(50)    NOT NULL,
    record_id       NUMBER(10)      NOT NULL,
    action          VARCHAR2(20)    NOT NULL,
    changed_by      NUMBER(10)      NOT NULL,
    changed_at      TIMESTAMP       DEFAULT SYSTIMESTAMP NOT NULL,
    old_value       CLOB,
    new_value       CLOB,
    CONSTRAINT pk_audit_logs    PRIMARY KEY (log_id),
    CONSTRAINT chk_audit_action CHECK (action IN ('INSERT','UPDATE','DELETE'))
);

-- ============================================================
-- INDEXES — critical for recursive CTE performance
-- ============================================================
CREATE INDEX idx_emp_manager_id    ON employees(manager_id);
CREATE INDEX idx_emp_department    ON employees(department_id);
CREATE INDEX idx_emp_status        ON employees(status);
CREATE INDEX idx_salary_employee   ON employee_salary_history(employee_id);
CREATE INDEX idx_salary_effective  ON employee_salary_history(employee_id, effective_to);
CREATE INDEX idx_approval_emp      ON approval_limits(employee_id);
```

---

## Sample Data

```sql
-- ============================================================
-- DEPARTMENTS
-- ============================================================
INSERT INTO departments VALUES (1, 'Executive',       'New York');
INSERT INTO departments VALUES (2, 'Engineering',     'San Francisco');
INSERT INTO departments VALUES (3, 'Sales',           'Chicago');
INSERT INTO departments VALUES (4, 'Human Resources', 'New York');
INSERT INTO departments VALUES (5, 'Finance',         'New York');

-- ============================================================
-- EMPLOYEES (5-level hierarchy)
-- Level 1: CEO
-- Level 2: Department Heads (report to CEO)
-- Level 3: Team Leads (report to Department Heads)
-- Level 4: Senior Staff (report to Team Leads)
-- Level 5: Junior Staff (report to Senior Staff)
-- ============================================================
-- Level 1
INSERT INTO employees VALUES (1, 'Alice Johnson',   'alice@company.com',      'CEO',                    1, NULL, DATE '2010-01-01', 'ACTIVE');
-- Level 2
INSERT INTO employees VALUES (2, 'Bob Smith',       'bob@company.com',         'VP Engineering',         2, 1,    DATE '2012-03-15', 'ACTIVE');
INSERT INTO employees VALUES (3, 'Carol Davis',     'carol@company.com',       'VP Sales',               3, 1,    DATE '2013-06-01', 'ACTIVE');
INSERT INTO employees VALUES (4, 'David Lee',       'david@company.com',       'VP HR',                  4, 1,    DATE '2014-02-10', 'ACTIVE');
INSERT INTO employees VALUES (5, 'Eve Martinez',    'eve@company.com',         'VP Finance',             5, 1,    DATE '2014-09-01', 'ACTIVE');
-- Level 3 — Under Engineering
INSERT INTO employees VALUES (6, 'Frank Wilson',    'frank@company.com',       'Backend Lead',           2, 2,    DATE '2015-01-10', 'ACTIVE');
INSERT INTO employees VALUES (7, 'Grace Kim',       'grace@company.com',       'Frontend Lead',          2, 2,    DATE '2015-04-01', 'ACTIVE');
INSERT INTO employees VALUES (8, 'Hank Brown',      'hank@company.com',        'DevOps Lead',            2, 2,    DATE '2016-07-01', 'ACTIVE');
-- Level 3 — Under Sales
INSERT INTO employees VALUES (9, 'Iris Chen',       'iris@company.com',        'Sales Lead East',        3, 3,    DATE '2016-01-15', 'ACTIVE');
INSERT INTO employees VALUES (10,'Jack Taylor',     'jack@company.com',        'Sales Lead West',        3, 3,    DATE '2016-05-01', 'ACTIVE');
-- Level 4 — Under Backend Lead
INSERT INTO employees VALUES (11,'Karen White',     'karen@company.com',       'Senior Backend Dev',     2, 6,    DATE '2017-03-01', 'ACTIVE');
INSERT INTO employees VALUES (12,'Leo Garcia',      'leo@company.com',         'Senior Backend Dev',     2, 6,    DATE '2017-08-15', 'ACTIVE');
-- Level 4 — Under Frontend Lead
INSERT INTO employees VALUES (13,'Mia Anderson',    'mia@company.com',         'Senior Frontend Dev',    2, 7,    DATE '2018-01-10', 'ACTIVE');
-- Level 4 — Under Sales Lead East
INSERT INTO employees VALUES (14,'Nate Thomas',     'nate@company.com',        'Senior Sales Rep',       3, 9,    DATE '2018-06-01', 'ACTIVE');
-- Level 5 — Under Senior Backend Dev (Karen)
INSERT INTO employees VALUES (15,'Olivia Harris',   'olivia@company.com',      'Junior Backend Dev',     2, 11,   DATE '2020-02-01', 'ACTIVE');
INSERT INTO employees VALUES (16,'Paul Clark',      'paul@company.com',        'Junior Backend Dev',     2, 11,   DATE '2020-09-01', 'ACTIVE');
-- Level 5 — Under Senior Sales Rep (Nate)
INSERT INTO employees VALUES (17,'Quinn Lewis',     'quinn@company.com',       'Sales Associate',        3, 14,   DATE '2021-03-15', 'ACTIVE');

COMMIT;

-- ============================================================
-- CURRENT SALARIES (effective_to IS NULL = current)
-- ============================================================
INSERT INTO employee_salary_history VALUES (1,  1,  250000, 'USD', DATE '2010-01-01', NULL);
INSERT INTO employee_salary_history VALUES (2,  2,  180000, 'USD', DATE '2012-03-15', NULL);
INSERT INTO employee_salary_history VALUES (3,  3,  160000, 'USD', DATE '2013-06-01', NULL);
INSERT INTO employee_salary_history VALUES (4,  4,  140000, 'USD', DATE '2014-02-10', NULL);
INSERT INTO employee_salary_history VALUES (5,  5,  155000, 'USD', DATE '2014-09-01', NULL);
INSERT INTO employee_salary_history VALUES (6,  6,  130000, 'USD', DATE '2015-01-10', NULL);
INSERT INTO employee_salary_history VALUES (7,  7,  125000, 'USD', DATE '2015-04-01', NULL);
INSERT INTO employee_salary_history VALUES (8,  8,  120000, 'USD', DATE '2016-07-01', NULL);
INSERT INTO employee_salary_history VALUES (9,  9,  110000, 'USD', DATE '2016-01-15', NULL);
INSERT INTO employee_salary_history VALUES (10, 10, 105000, 'USD', DATE '2016-05-01', NULL);
INSERT INTO employee_salary_history VALUES (11, 11,  95000, 'USD', DATE '2017-03-01', NULL);
INSERT INTO employee_salary_history VALUES (12, 12,  92000, 'USD', DATE '2017-08-15', NULL);
INSERT INTO employee_salary_history VALUES (13, 13,  90000, 'USD', DATE '2018-01-10', NULL);
INSERT INTO employee_salary_history VALUES (14, 14,  85000, 'USD', DATE '2018-06-01', NULL);
INSERT INTO employee_salary_history VALUES (15, 15,  70000, 'USD', DATE '2020-02-01', NULL);
INSERT INTO employee_salary_history VALUES (16, 16,  68000, 'USD', DATE '2020-09-01', NULL);
INSERT INTO employee_salary_history VALUES (17, 17,  55000, 'USD', DATE '2021-03-15', NULL);

-- ============================================================
-- APPROVAL LIMITS
-- ============================================================
INSERT INTO approval_limits VALUES (1, 1, 'EXPENSE',  500000);
INSERT INTO approval_limits VALUES (2, 1, 'HIRING',   500000);
INSERT INTO approval_limits VALUES (3, 2, 'EXPENSE',  100000);
INSERT INTO approval_limits VALUES (4, 3, 'EXPENSE',   80000);
INSERT INTO approval_limits VALUES (5, 6, 'EXPENSE',   20000);
INSERT INTO approval_limits VALUES (6, 9, 'EXPENSE',   10000);
INSERT INTO approval_limits VALUES (7, 11,'EXPENSE',    5000);

COMMIT;
```

---

## Recursive CTE Examples

### Example 1: Complete Organization Tree

This query traverses the entire employee hierarchy from CEO down to the deepest level, building a hierarchy path and visual indentation.

```sql
-- ============================================================
-- COMPLETE ORGANIZATION TREE
-- Shows every employee from CEO (level 1) to deepest staff
-- ============================================================
WITH emp_hierarchy (
    employee_id,
    employee_name,
    job_title,
    manager_id,
    department_id,
    hierarchy_level,
    hierarchy_path,
    root_manager_id,
    indent_name
) AS (
    -- ---- ANCHOR QUERY ----
    -- Start from the CEO (manager_id IS NULL)
    SELECT
        e.employee_id,
        e.employee_name,
        e.job_title,
        e.manager_id,
        e.department_id,
        1                                          AS hierarchy_level,
        CAST(e.employee_name AS VARCHAR2(4000))    AS hierarchy_path,
        e.employee_id                              AS root_manager_id,
        e.employee_name                            AS indent_name
    FROM employees e
    WHERE e.manager_id IS NULL
      AND e.status = 'ACTIVE'

    UNION ALL

    -- ---- RECURSIVE QUERY ----
    -- For each employee already found, find their direct reports
    SELECT
        e.employee_id,
        e.employee_name,
        e.job_title,
        e.manager_id,
        e.department_id,
        h.hierarchy_level + 1,
        CAST(h.hierarchy_path || ' > ' || e.employee_name AS VARCHAR2(4000)),
        h.root_manager_id,
        CAST(LPAD(' ', (h.hierarchy_level) * 4, ' ') || e.employee_name AS VARCHAR2(4000))
    FROM employees e
    JOIN emp_hierarchy h ON e.manager_id = h.employee_id
    WHERE e.status = 'ACTIVE'
)
-- ---- MAIN QUERY ----
SELECT
    h.hierarchy_level                          AS "Level",
    h.indent_name                              AS "Employee (Indented)",
    h.job_title                                AS "Job Title",
    d.department_name                          AS "Department",
    mgr.employee_name                          AS "Reports To",
    h.hierarchy_path                           AS "Full Path"
FROM emp_hierarchy h
JOIN departments d ON h.department_id = d.department_id
LEFT JOIN employees mgr ON h.manager_id = mgr.employee_id
ORDER BY h.hierarchy_path;
```

**Step-by-Step Explanation:**

- **Anchor query**: Selects the CEO (`manager_id IS NULL`). Seeds `hierarchy_level = 1`, `hierarchy_path` = CEO's name, and `root_manager_id` = CEO's ID.
- **Recursive query**: For each already-found employee, finds their direct reports by joining `employees` to the CTE on `e.manager_id = h.employee_id`. Increments the level, concatenates the path with ` > `.
- **`LPAD(' ', (h.hierarchy_level) * 4, ' ')`**: Creates visual indentation — 4 spaces per level — to show tree structure.
- **`CAST(... AS VARCHAR2(4000))`**: Required in Oracle. The recursive query must produce the same data types as the anchor. Since string concatenation can overflow VARCHAR2, we cast explicitly and choose a safe length.
- **Main query**: Joins with `departments` for department name and `employees` again to resolve manager name.

**Expected Output (partial):**

| Level | Employee (Indented) | Job Title | Department | Reports To | Full Path |
|---|---|---|---|---|---|
| 1 | Alice Johnson | CEO | Executive | _(null)_ | Alice Johnson |
| 2 | ····Bob Smith | VP Engineering | Engineering | Alice Johnson | Alice Johnson > Bob Smith |
| 2 | ····Carol Davis | VP Sales | Sales | Alice Johnson | Alice Johnson > Carol Davis |
| 3 | ········Frank Wilson | Backend Lead | Engineering | Bob Smith | Alice Johnson > Bob Smith > Frank Wilson |
| 4 | ············Karen White | Senior Backend Dev | Engineering | Frank Wilson | Alice Johnson > Bob Smith > Frank Wilson > Karen White |
| 5 | ················Olivia Harris | Junior Backend Dev | Engineering | Karen White | Alice Johnson > Bob Smith > Frank Wilson > Karen White > Olivia Harris |

---

### Example 2: All Subordinates Under a Manager

Given a specific manager's ID, find all employees who report to them, directly or indirectly, at any depth.

```sql
-- ============================================================
-- ALL SUBORDINATES UNDER A GIVEN MANAGER
-- Replace :manager_id with any employee_id (e.g., 2 = Bob Smith)
-- ============================================================
WITH subordinates (
    employee_id,
    employee_name,
    job_title,
    manager_id,
    hierarchy_level,
    hierarchy_path
) AS (
    -- ANCHOR: the manager themselves
    SELECT
        e.employee_id,
        e.employee_name,
        e.job_title,
        e.manager_id,
        0                                        AS hierarchy_level,  -- 0 = the root manager
        CAST(e.employee_name AS VARCHAR2(4000))  AS hierarchy_path
    FROM employees e
    WHERE e.employee_id = :manager_id           -- bind parameter

    UNION ALL

    -- RECURSIVE: find all employees who report under the current level
    SELECT
        e.employee_id,
        e.employee_name,
        e.job_title,
        e.manager_id,
        s.hierarchy_level + 1,
        CAST(s.hierarchy_path || ' > ' || e.employee_name AS VARCHAR2(4000))
    FROM employees e
    JOIN subordinates s ON e.manager_id = s.employee_id
    WHERE e.status = 'ACTIVE'
)
SELECT
    s.hierarchy_level              AS "Depth From Manager",
    s.employee_name                AS "Employee",
    s.job_title                    AS "Title",
    s.hierarchy_path               AS "Path",
    CASE
        WHEN NOT EXISTS (
            SELECT 1 FROM employees sub
            WHERE sub.manager_id = s.employee_id
              AND sub.status = 'ACTIVE'
        ) THEN 'Yes'
        ELSE 'No'
    END                            AS "Is Leaf Node?"
FROM subordinates s
WHERE s.hierarchy_level > 0        -- exclude the manager themselves
ORDER BY s.hierarchy_path;
```

**What this does:**

- **Anchor at `hierarchy_level = 0`**: Seeds the manager as the starting point without including them in final output (filtered by `WHERE hierarchy_level > 0`).
- **Recursive step**: Keeps finding children of children until no more rows are returned.
- **`CASE ... NOT EXISTS`**: Identifies leaf nodes (employees with no direct reports).

**Expected Output for `manager_id = 2` (Bob Smith):**

| Depth From Manager | Employee | Title | Path | Is Leaf Node? |
|---|---|---|---|---|
| 1 | Frank Wilson | Backend Lead | Bob Smith > Frank Wilson | No |
| 1 | Grace Kim | Frontend Lead | Bob Smith > Grace Kim | No |
| 1 | Hank Brown | DevOps Lead | Bob Smith > Hank Brown | Yes |
| 2 | Karen White | Senior Backend Dev | Bob Smith > Frank Wilson > Karen White | No |
| 2 | Leo Garcia | Senior Backend Dev | Bob Smith > Frank Wilson > Leo Garcia | Yes |
| 2 | Mia Anderson | Senior Frontend Dev | Bob Smith > Grace Kim > Mia Anderson | Yes |
| 3 | Olivia Harris | Junior Backend Dev | Bob Smith > Frank Wilson > Karen White > Olivia Harris | Yes |
| 3 | Paul Clark | Junior Backend Dev | Bob Smith > Frank Wilson > Karen White > Paul Clark | Yes |

---

### Example 3: Approval Chain Upward to CEO

Given an employee's ID, find the complete chain of authority from that employee up to the CEO, along with each approver's spending limit.

```sql
-- ============================================================
-- APPROVAL CHAIN — FROM EMPLOYEE UP TO CEO
-- Replace :employee_id with the starting employee (e.g., 15)
-- ============================================================
WITH approval_chain (
    employee_id,
    employee_name,
    job_title,
    manager_id,
    chain_step,
    chain_path
) AS (
    -- ANCHOR: start from the given employee
    SELECT
        e.employee_id,
        e.employee_name,
        e.job_title,
        e.manager_id,
        1                                        AS chain_step,
        CAST(e.employee_name AS VARCHAR2(4000))  AS chain_path
    FROM employees e
    WHERE e.employee_id = :employee_id

    UNION ALL

    -- RECURSIVE: walk up to manager, then manager's manager, etc.
    SELECT
        e.employee_id,
        e.employee_name,
        e.job_title,
        e.manager_id,
        ac.chain_step + 1,
        CAST(ac.chain_path || ' → ' || e.employee_name AS VARCHAR2(4000))
    FROM employees e
    JOIN approval_chain ac ON e.employee_id = ac.manager_id  -- NOTE: reversed join — child references parent
)
SELECT
    ac.chain_step          AS "Step",
    ac.employee_name       AS "Approver",
    ac.job_title           AS "Title",
    al.max_amount          AS "Expense Limit (USD)",
    ac.chain_path          AS "Chain So Far"
FROM approval_chain ac
LEFT JOIN approval_limits al
       ON al.employee_id = ac.employee_id
      AND al.limit_type = 'EXPENSE'
ORDER BY ac.chain_step;
```

**Key difference from top-down traversal:**

- In a **bottom-up** recursive CTE, the recursive join is **reversed**: `JOIN approval_chain ac ON e.employee_id = ac.manager_id`. This means: "find the employee whose ID matches the current row's `manager_id`" — walking upward instead of downward.

**Expected Output for `employee_id = 15` (Olivia Harris):**

| Step | Approver | Title | Expense Limit (USD) | Chain So Far |
|---|---|---|---|---|
| 1 | Olivia Harris | Junior Backend Dev | _(none)_ | Olivia Harris |
| 2 | Karen White | Senior Backend Dev | 5,000 | Olivia Harris → Karen White |
| 3 | Frank Wilson | Backend Lead | 20,000 | Olivia Harris → Karen White → Frank Wilson |
| 4 | Bob Smith | VP Engineering | 100,000 | Olivia Harris → Karen White → Frank Wilson → Bob Smith |
| 5 | Alice Johnson | CEO | 500,000 | Olivia Harris → Karen White → Frank Wilson → Bob Smith → Alice Johnson |

---

### Example 4: Salary Roll-Up Per Manager

Calculate how much salary cost each manager is responsible for — including all direct and indirect subordinates — then join with salary history.

```sql
-- ============================================================
-- SALARY ROLL-UP — TOTAL SALARY COST UNDER EACH MANAGER
-- ============================================================
WITH emp_tree (
    root_manager_id,
    employee_id,
    manager_id,
    hierarchy_level
) AS (
    -- ANCHOR: every active employee is a root of their own subtree
    SELECT
        e.employee_id  AS root_manager_id,
        e.employee_id,
        e.manager_id,
        0              AS hierarchy_level
    FROM employees e
    WHERE e.status = 'ACTIVE'

    UNION ALL

    -- RECURSIVE: find all reports under each root
    SELECT
        t.root_manager_id,
        e.employee_id,
        e.manager_id,
        t.hierarchy_level + 1
    FROM employees e
    JOIN emp_tree t ON e.manager_id = t.employee_id
    WHERE e.status = 'ACTIVE'
),
-- Current salaries only
current_salaries AS (
    SELECT employee_id, salary_amount
    FROM employee_salary_history
    WHERE effective_to IS NULL
)
SELECT
    mgr.employee_id                                     AS "Manager ID",
    mgr.employee_name                                   AS "Manager Name",
    mgr.job_title                                       AS "Title",
    d.department_name                                   AS "Department",
    COUNT(DISTINCT t.employee_id) - 1                   AS "Total Reports (All Levels)",
    SUM(cs.salary_amount)                               AS "Total Salary Under Manager (USD)"
FROM emp_tree t
JOIN employees mgr ON t.root_manager_id = mgr.employee_id
JOIN employees e   ON t.employee_id = e.employee_id
JOIN departments d ON mgr.department_id = d.department_id
JOIN current_salaries cs ON t.employee_id = cs.employee_id
WHERE mgr.status = 'ACTIVE'
GROUP BY
    mgr.employee_id,
    mgr.employee_name,
    mgr.job_title,
    d.department_name
HAVING COUNT(DISTINCT t.employee_id) > 1              -- only show managers (have at least 1 report)
ORDER BY SUM(cs.salary_amount) DESC;
```

**How this works:**

- The CTE starts from every active employee as a `root_manager_id`.
- The recursive step finds all employees under each root.
- After recursion, the main query groups by `root_manager_id`, summing all salaries below each manager (including the manager themselves — hence `COUNT - 1` for "reports only").
- `HAVING COUNT > 1` filters out employees who manage no one.

**Expected Output (selected rows):**

| Manager ID | Manager Name | Title | Department | Total Reports | Total Salary Under Manager |
|---|---|---|---|---|---|
| 1 | Alice Johnson | CEO | Executive | 16 | 2,230,000 |
| 2 | Bob Smith | VP Engineering | Engineering | 9 | 1,090,000 |
| 6 | Frank Wilson | Backend Lead | Engineering | 4 | 430,000 |
| 3 | Carol Davis | VP Sales | Sales | 4 | 415,000 |
| 11 | Karen White | Senior Backend Dev | Engineering | 2 | 208,000 |

---

### Example 5: Leaf Nodes — Employees Without Subordinates

```sql
-- ============================================================
-- LEAF NODES: EMPLOYEES WITH NO DIRECT REPORTS
-- ============================================================
WITH emp_hierarchy (
    employee_id,
    employee_name,
    job_title,
    manager_id,
    hierarchy_level
) AS (
    SELECT e.employee_id, e.employee_name, e.job_title, e.manager_id, 1
    FROM employees e
    WHERE e.manager_id IS NULL AND e.status = 'ACTIVE'

    UNION ALL

    SELECT e.employee_id, e.employee_name, e.job_title, e.manager_id, h.hierarchy_level + 1
    FROM employees e
    JOIN emp_hierarchy h ON e.manager_id = h.employee_id
    WHERE e.status = 'ACTIVE'
)
SELECT
    h.hierarchy_level   AS "Level",
    h.employee_name     AS "Employee",
    h.job_title         AS "Title"
FROM emp_hierarchy h
WHERE h.employee_id NOT IN (
    SELECT DISTINCT manager_id
    FROM employees
    WHERE manager_id IS NOT NULL
      AND status = 'ACTIVE'
)
ORDER BY h.hierarchy_level, h.employee_name;
```

**Expected Output:**

| Level | Employee | Title |
|---|---|---|
| 3 | Hank Brown | DevOps Lead |
| 4 | Leo Garcia | Senior Backend Dev |
| 4 | Mia Anderson | Senior Frontend Dev |
| 4 | Nate Thomas | Senior Sales Rep |
| 5 | Olivia Harris | Junior Backend Dev |
| 5 | Paul Clark | Junior Backend Dev |
| 5 | Quinn Lewis | Sales Associate |

---

### Example 6: Distance Between Two Employees

Find how many levels separate two employees in the hierarchy.

```sql
-- ============================================================
-- DISTANCE BETWEEN TWO EMPLOYEES IN THE HIERARCHY
-- Find the common ancestor, then measure distance
-- Replace :emp_a and :emp_b with employee IDs
-- ============================================================
WITH
-- Build path to root for Employee A
path_a (employee_id, manager_id, depth) AS (
    SELECT employee_id, manager_id, 0
    FROM employees WHERE employee_id = :emp_a

    UNION ALL

    SELECT e.employee_id, e.manager_id, p.depth + 1
    FROM employees e
    JOIN path_a p ON e.employee_id = p.manager_id
),
-- Build path to root for Employee B
path_b (employee_id, manager_id, depth) AS (
    SELECT employee_id, manager_id, 0
    FROM employees WHERE employee_id = :emp_b

    UNION ALL

    SELECT e.employee_id, e.manager_id, p.depth + 1
    FROM employees e
    JOIN path_b p ON e.employee_id = p.manager_id
)
-- Find lowest common ancestor and compute total distance
SELECT
    a.employee_id                      AS "Common Ancestor ID",
    e.employee_name                    AS "Common Ancestor",
    a.depth                            AS "Distance from Emp A",
    b.depth                            AS "Distance from Emp B",
    a.depth + b.depth                  AS "Total Distance"
FROM path_a a
JOIN path_b b ON a.employee_id = b.employee_id
JOIN employees e ON a.employee_id = e.employee_id
ORDER BY a.depth + b.depth
FETCH FIRST 1 ROW ONLY;  -- closest common ancestor
```

---

### Example 7: Cycle Detection

If hierarchy data is user-managed (e.g., HR enters data manually), circular references can accidentally occur. This example detects them.

```sql
-- ============================================================
-- CYCLE DETECTION — Find circular reporting relationships
-- Uses a depth counter as a safety brake
-- ============================================================
WITH emp_hierarchy (
    employee_id,
    employee_name,
    manager_id,
    depth,
    visited_ids,
    is_cycle
) AS (
    -- ANCHOR
    SELECT
        e.employee_id,
        e.employee_name,
        e.manager_id,
        0                                        AS depth,
        CAST(',' || e.employee_id || ',' AS VARCHAR2(4000)) AS visited_ids,
        0                                        AS is_cycle
    FROM employees e
    WHERE e.manager_id IS NULL

    UNION ALL

    -- RECURSIVE with cycle detection
    SELECT
        e.employee_id,
        e.employee_name,
        e.manager_id,
        h.depth + 1,
        CAST(h.visited_ids || e.employee_id || ',' AS VARCHAR2(4000)),
        CASE
            WHEN INSTR(h.visited_ids, ',' || e.employee_id || ',') > 0 THEN 1
            ELSE 0
        END
    FROM employees e
    JOIN emp_hierarchy h ON e.manager_id = h.employee_id
    WHERE h.is_cycle = 0      -- stop recursing if a cycle was already found
      AND h.depth < 20        -- hard depth limit as safety brake
)
SELECT
    employee_id,
    employee_name,
    depth,
    visited_ids
FROM emp_hierarchy
WHERE is_cycle = 1;
```

**How cycle detection works:**

- `visited_ids` accumulates a comma-delimited string of all employee IDs seen along the current path (e.g., `,1,2,6,`).
- Before adding the next employee, `INSTR(h.visited_ids, ',' || e.employee_id || ',')` checks whether the ID already appears in the path.
- If found, `is_cycle = 1` is set and the recursive step stops (`WHERE h.is_cycle = 0`).
- A hard `depth < 20` limit is added as a secondary safety brake.

---

## Product Category Tree Example

This example shows how a Laravel e-commerce application can use Recursive CTE to build a nested product category menu.

### Table Design

```sql
CREATE TABLE product_categories (
    category_id     NUMBER(10)      NOT NULL,
    category_name   VARCHAR2(100)   NOT NULL,
    parent_id       NUMBER(10),
    slug            VARCHAR2(150)   NOT NULL,
    is_active       NUMBER(1)       DEFAULT 1 NOT NULL,
    sort_order      NUMBER(5)       DEFAULT 0 NOT NULL,
    CONSTRAINT pk_categories   PRIMARY KEY (category_id),
    CONSTRAINT fk_cat_parent   FOREIGN KEY (parent_id)
        REFERENCES product_categories(category_id),
    CONSTRAINT chk_cat_active  CHECK (is_active IN (0, 1))
);

CREATE INDEX idx_cat_parent ON product_categories(parent_id);

-- Sample data
INSERT INTO product_categories VALUES (1,  'Electronics',      NULL, 'electronics',       1, 1);
INSERT INTO product_categories VALUES (2,  'Mobile Phones',    1,    'mobile-phones',      1, 1);
INSERT INTO product_categories VALUES (3,  'Laptops',          1,    'laptops',            1, 2);
INSERT INTO product_categories VALUES (4,  'Accessories',      1,    'accessories',        1, 3);
INSERT INTO product_categories VALUES (5,  'Android Phones',   2,    'android-phones',     1, 1);
INSERT INTO product_categories VALUES (6,  'iPhones',          2,    'iphones',            1, 2);
INSERT INTO product_categories VALUES (7,  'Gaming Laptops',   3,    'gaming-laptops',     1, 1);
INSERT INTO product_categories VALUES (8,  'Business Laptops', 3,    'business-laptops',   1, 2);
INSERT INTO product_categories VALUES (9,  'Samsung',          5,    'samsung',            1, 1);
INSERT INTO product_categories VALUES (10, 'Google Pixel',     5,    'google-pixel',       1, 2);
INSERT INTO product_categories VALUES (11, 'iPhone 15',        6,    'iphone-15',          1, 1);
INSERT INTO product_categories VALUES (12, 'iPhone 16',        6,    'iphone-16',          1, 2);
COMMIT;
```

### Recursive CTE for Full Category Tree

```sql
-- ============================================================
-- FULL PRODUCT CATEGORY TREE
-- ============================================================
WITH category_tree (
    category_id,
    category_name,
    parent_id,
    slug,
    depth,
    full_path,
    breadcrumb,
    sort_path
) AS (
    -- ANCHOR: top-level categories (no parent)
    SELECT
        c.category_id,
        c.category_name,
        c.parent_id,
        c.slug,
        0                                         AS depth,
        CAST(c.category_name AS VARCHAR2(4000))   AS full_path,
        CAST(c.category_name AS VARCHAR2(4000))   AS breadcrumb,
        CAST(LPAD(c.sort_order, 5, '0') AS VARCHAR2(4000)) AS sort_path
    FROM product_categories c
    WHERE c.parent_id IS NULL
      AND c.is_active = 1

    UNION ALL

    -- RECURSIVE: children of each found category
    SELECT
        c.category_id,
        c.category_name,
        c.parent_id,
        c.slug,
        t.depth + 1,
        CAST(t.full_path || ' / ' || c.category_name AS VARCHAR2(4000)),
        CAST(t.breadcrumb || ' > ' || c.category_name AS VARCHAR2(4000)),
        CAST(t.sort_path || '-' || LPAD(c.sort_order, 5, '0') AS VARCHAR2(4000))
    FROM product_categories c
    JOIN category_tree t ON c.parent_id = t.category_id
    WHERE c.is_active = 1
)
SELECT
    ct.depth                                AS "Level",
    LPAD('-', ct.depth * 2, '-') ||
        ct.category_name                    AS "Category (Tree View)",
    ct.slug                                 AS "Slug",
    ct.breadcrumb                           AS "Breadcrumb",
    ct.full_path                            AS "Full Path"
FROM category_tree ct
ORDER BY ct.sort_path;
```

**Expected Output:**

| Level | Category (Tree View) | Slug | Breadcrumb |
|---|---|---|---|
| 0 | Electronics | electronics | Electronics |
| 1 | --Mobile Phones | mobile-phones | Electronics > Mobile Phones |
| 2 | ----Android Phones | android-phones | Electronics > Mobile Phones > Android Phones |
| 3 | ------Samsung | samsung | Electronics > Mobile Phones > Android Phones > Samsung |
| 3 | ------Google Pixel | google-pixel | Electronics > Mobile Phones > Android Phones > Google Pixel |
| 2 | ----iPhones | iphones | Electronics > Mobile Phones > iPhones |
| 3 | ------iPhone 15 | iphone-15 | Electronics > Mobile Phones > iPhones > iPhone 15 |
| 3 | ------iPhone 16 | iphone-16 | Electronics > Mobile Phones > iPhones > iPhone 16 |
| 1 | --Laptops | laptops | Electronics > Laptops |
| 2 | ----Gaming Laptops | gaming-laptops | Electronics > Laptops > Gaming Laptops |
| 2 | ----Business Laptops | business-laptops | Electronics > Laptops > Business Laptops |

---

## Recursive CTE vs CONNECT BY — Detailed Comparison

### Side-by-Side Feature Comparison

| Feature | Recursive CTE | CONNECT BY |
|---|---|---|
| SQL Standard | ANSI SQL:1999 ✅ | Oracle-proprietary ❌ |
| Oracle version | 11g R2+ | Since Oracle 2 (very old) |
| Portability | PostgreSQL, SQL Server, DB2, MySQL 8+ | Oracle only |
| Readability | More verbose, self-documenting | Concise, Oracle-idiomatic |
| Complex JOINs in recursion | Fully supported | Limited and awkward |
| Multiple tables in recursion | Easy | Cumbersome |
| Aggregation during traversal | Supported | Not directly |
| Built-in `LEVEL` pseudo-column | Not available (use manual counter) | `LEVEL` available |
| Path generation | Manual string concatenation | `SYS_CONNECT_BY_PATH()` |
| Root node | Manual `WHERE` + `UNION ALL` | `START WITH` clause |
| Cycle detection | Manual (counter / string check) | `NOCYCLE` keyword |
| Bottom-up traversal | Straightforward reversed join | Awkward |
| `PRIOR` keyword | Not used (use explicit join) | `PRIOR` required |
| Passing values down levels | Easy through CTE columns | `CONNECT_BY_ROOT` for root value only |
| Filter at each level | Full SQL flexibility | Limited |
| Modern Oracle development | Recommended | Legacy |

### Recursive CTE — Employee Hierarchy Example

```sql
-- RECURSIVE CTE APPROACH
WITH emp_hierarchy (employee_id, employee_name, manager_id, lvl, path) AS (
    SELECT employee_id, employee_name, manager_id, 1,
           CAST(employee_name AS VARCHAR2(4000))
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    SELECT e.employee_id, e.employee_name, e.manager_id, h.lvl + 1,
           CAST(h.path || ' > ' || e.employee_name AS VARCHAR2(4000))
    FROM employees e
    JOIN emp_hierarchy h ON e.manager_id = h.employee_id
)
SELECT lvl, employee_name, path
FROM emp_hierarchy
ORDER BY path;
```

### CONNECT BY — Same Query

```sql
-- CONNECT BY APPROACH
SELECT
    LEVEL                                    AS lvl,
    employee_name,
    SYS_CONNECT_BY_PATH(employee_name, ' > ') AS path
FROM employees
START WITH manager_id IS NULL
CONNECT BY PRIOR employee_id = manager_id
ORDER SIBLINGS BY employee_name;
```

### When to Choose Which

**Choose Recursive CTE when:**
- Your application may migrate to another database platform.
- The recursion involves complex joins across 2+ tables (e.g., joining salary history inside the recursion).
- You need to pass computed values from parent to child (e.g., accumulated budget).
- You are doing bottom-up traversal and need clean, readable code.
- Your team is writing modern SQL and you want code that is standard-compliant.

**Choose CONNECT BY when:**
- You are writing Oracle-only queries and want brevity.
- You need `LEVEL`, `CONNECT_BY_ROOT`, `CONNECT_BY_ISLEAF`, or `SYS_CONNECT_BY_PATH` without building them manually.
- The hierarchy is simple and lives in a single table.
- You are maintaining or extending existing legacy Oracle code.

---

## Performance Tuning

### Why Recursive CTE Can Become Slow

| Cause | Impact | Solution |
|---|---|---|
| Missing index on `manager_id` | Full table scan every iteration | `CREATE INDEX idx_emp_manager ON employees(manager_id)` |
| Missing index on `parent_id` | Same as above | Index every parent-child column |
| Too many recursion levels | Long execution, memory pressure | Add `WHERE depth < :max_depth` |
| Circular references | Infinite recursion / crash | Add cycle detection |
| Unnecessary columns in CTE | Larger working set per iteration | Select only needed columns |
| Filtering after recursion | All rows generated before filter applied | Push filters into anchor/recursive parts |
| Large path strings | Memory-intensive `VARCHAR2(4000)` concatenation | Limit path length or use rowid tracking instead |
| Using `UNION` instead of `UNION ALL` | Sort + dedup on every iteration | Always use `UNION ALL` |
| Too many CTEs chained | Optimizer materializes intermediate sets | Minimize CTE count; use hints if needed |

### Recommended Index Strategy

```sql
-- Most important: index on the join column used in recursive step
CREATE INDEX idx_emp_manager_id   ON employees(manager_id);
CREATE INDEX idx_cat_parent_id    ON product_categories(parent_id);

-- Composite index if you frequently filter by status AND manager
CREATE INDEX idx_emp_mgr_status   ON employees(manager_id, status);

-- Covering index if you select name and job_title frequently
CREATE INDEX idx_emp_mgr_cover    ON employees(manager_id, employee_id, employee_name, job_title);
```

### Limiting Recursion Depth

Always add a depth guard for user-managed hierarchy data:

```sql
WITH emp_hierarchy (employee_id, manager_id, depth) AS (
    SELECT employee_id, manager_id, 0
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    SELECT e.employee_id, e.manager_id, h.depth + 1
    FROM employees e
    JOIN emp_hierarchy h ON e.manager_id = h.employee_id
    WHERE h.depth < 50    -- HARD LIMIT: stops at 50 levels regardless
)
SELECT * FROM emp_hierarchy;
```

### Analyzing Recursive Query Performance

Use `EXPLAIN PLAN` to understand how Oracle executes the recursive CTE:

```sql
EXPLAIN PLAN FOR
WITH emp_hierarchy (employee_id, employee_name, manager_id, lvl) AS (
    SELECT employee_id, employee_name, manager_id, 1
    FROM employees WHERE manager_id IS NULL

    UNION ALL

    SELECT e.employee_id, e.employee_name, e.manager_id, h.lvl + 1
    FROM employees e
    JOIN emp_hierarchy h ON e.manager_id = h.employee_id
)
SELECT * FROM emp_hierarchy;

-- Read the plan
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

**Key things to look for in the plan:**

- `RECURSIVE WITH PUMP` — Oracle's internal iterator for recursive CTE. This is normal.
- `TABLE ACCESS FULL` on the employees table — means missing index on `manager_id`. Add the index.
- `HASH JOIN` vs `NESTED LOOPS` — for small data sets, Nested Loops is preferred. For large ones, Hash Join can be faster.
- `TEMP TABLE TRANSFORMATION` — Oracle may materialize the CTE. This is expected for recursive CTEs.

### Using the `SEARCH` Clause for Ordering

Oracle supports an optional `SEARCH` clause to control the order of recursive traversal:

```sql
WITH emp_hierarchy (employee_id, employee_name, manager_id, lvl) AS (
    SELECT employee_id, employee_name, manager_id, 1
    FROM employees WHERE manager_id IS NULL

    UNION ALL

    SELECT e.employee_id, e.employee_name, e.manager_id, h.lvl + 1
    FROM employees e
    JOIN emp_hierarchy h ON e.manager_id = h.employee_id
)
SEARCH DEPTH FIRST BY employee_name SET traversal_order   -- depth-first ordering
SELECT employee_id, employee_name, lvl, traversal_order
FROM emp_hierarchy
ORDER BY traversal_order;
```

- `SEARCH DEPTH FIRST BY ... SET order_col` — processes each branch completely before moving to the next (like DFS).
- `SEARCH BREADTH FIRST BY ... SET order_col` — processes all nodes at the same level before going deeper (like BFS).

### Using the `CYCLE` Clause

Oracle has a built-in `CYCLE` clause that automatically detects cycles:

```sql
WITH emp_hierarchy (employee_id, employee_name, manager_id) AS (
    SELECT employee_id, employee_name, manager_id
    FROM employees WHERE manager_id IS NULL

    UNION ALL

    SELECT e.employee_id, e.employee_name, e.manager_id
    FROM employees e
    JOIN emp_hierarchy h ON e.manager_id = h.employee_id
)
CYCLE employee_id SET is_cycle TO '1' DEFAULT '0'
SELECT employee_id, employee_name, is_cycle
FROM emp_hierarchy
WHERE is_cycle = '1';
```

- `CYCLE employee_id` — tells Oracle which column to monitor for repeated values.
- `SET is_cycle TO '1' DEFAULT '0'` — adds a marker column to identify cyclic rows.
- Oracle automatically stops recursion when a cycle is detected.

---

## Laravel Developer Integration

### Why Laravel Developers Should Understand Recursive CTE

Laravel's Eloquent ORM is excellent for simple CRUD and even moderately complex queries. However, hierarchical data is where Eloquent falls short:

- Eloquent has no built-in recursive query support for Oracle.
- Building a tree in PHP from flat database rows requires recursive PHP functions, which become slow and memory-intensive for large trees.
- Running a separate query per tree level (N+1 problem) kills performance.
- A single well-written Oracle Recursive CTE returns the **complete tree in one query**, ready to consume in PHP.

### Practical Laravel Use Cases

- **Category menu generation** — nested menus for product categories.
- **Employee hierarchy** — organization charts, reports by department.
- **Approval workflow** — find next approver, find full approval chain.
- **Referral system** — count how many referrals exist under each referrer.
- **Comment/reply tree** — load nested comments in a forum or blog post.
- **Multi-level permissions** — role inheritance trees.
- **Organization chart** — building an interactive org chart API response.

### Laravel Example: Full Hierarchy Query Using `DB::select()`

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\DB;

class EmployeeHierarchyService
{
    /**
     * Get the complete organization hierarchy from CEO downward.
     *
     * @return array
     */
    public function getFullHierarchy(): array
    {
        $sql = "
            WITH emp_hierarchy (
                employee_id,
                employee_name,
                job_title,
                manager_id,
                department_id,
                hierarchy_level,
                hierarchy_path
            ) AS (
                SELECT
                    e.employee_id,
                    e.employee_name,
                    e.job_title,
                    e.manager_id,
                    e.department_id,
                    1 AS hierarchy_level,
                    CAST(e.employee_name AS VARCHAR2(4000)) AS hierarchy_path
                FROM employees e
                WHERE e.manager_id IS NULL
                  AND e.status = 'ACTIVE'

                UNION ALL

                SELECT
                    e.employee_id,
                    e.employee_name,
                    e.job_title,
                    e.manager_id,
                    e.department_id,
                    h.hierarchy_level + 1,
                    CAST(h.hierarchy_path || ' > ' || e.employee_name AS VARCHAR2(4000))
                FROM employees e
                JOIN emp_hierarchy h ON e.manager_id = h.employee_id
                WHERE e.status = 'ACTIVE'
            )
            SELECT
                h.employee_id,
                h.employee_name,
                h.job_title,
                h.hierarchy_level,
                h.hierarchy_path,
                d.department_name
            FROM emp_hierarchy h
            JOIN departments d ON h.department_id = d.department_id
            ORDER BY h.hierarchy_path
        ";

        $rows = DB::select($sql);

        return $this->buildTree($rows);
    }

    /**
     * Build a nested tree array from flat hierarchy rows.
     *
     * @param array $rows
     * @return array
     */
    private function buildTree(array $rows): array
    {
        $tree  = [];
        $index = [];

        foreach ($rows as $row) {
            $node = [
                'employee_id'    => $row->employee_id,
                'employee_name'  => $row->employee_name,
                'job_title'      => $row->job_title,
                'department'     => $row->department_name,
                'level'          => $row->hierarchy_level,
                'path'           => $row->hierarchy_path,
                'children'       => [],
            ];

            $index[$row->employee_id] = &$node;

            // The flat rows come ordered by path, so parent always appears first
            if ($row->hierarchy_level === 1) {
                $tree[] = &$node;
            }

            unset($node);
        }

        return $tree;
    }

    /**
     * Get all direct and indirect subordinates under a given manager.
     *
     * @param int $managerId
     * @return array
     */
    public function getSubordinates(int $managerId): array
    {
        $sql = "
            WITH subordinates (
                employee_id,
                employee_name,
                job_title,
                manager_id,
                hierarchy_level,
                hierarchy_path
            ) AS (
                SELECT
                    e.employee_id,
                    e.employee_name,
                    e.job_title,
                    e.manager_id,
                    0 AS hierarchy_level,
                    CAST(e.employee_name AS VARCHAR2(4000)) AS hierarchy_path
                FROM employees e
                WHERE e.employee_id = :manager_id

                UNION ALL

                SELECT
                    e.employee_id,
                    e.employee_name,
                    e.job_title,
                    e.manager_id,
                    s.hierarchy_level + 1,
                    CAST(s.hierarchy_path || ' > ' || e.employee_name AS VARCHAR2(4000))
                FROM employees e
                JOIN subordinates s ON e.manager_id = s.employee_id
                WHERE e.status = 'ACTIVE'
            )
            SELECT
                employee_id,
                employee_name,
                job_title,
                hierarchy_level,
                hierarchy_path
            FROM subordinates
            WHERE hierarchy_level > 0
            ORDER BY hierarchy_path
        ";

        return DB::select($sql, ['manager_id' => $managerId]);
    }

    /**
     * Get the approval chain from an employee up to the CEO.
     *
     * @param int $employeeId
     * @return array
     */
    public function getApprovalChain(int $employeeId): array
    {
        $sql = "
            WITH approval_chain (
                employee_id,
                employee_name,
                job_title,
                manager_id,
                chain_step
            ) AS (
                SELECT
                    e.employee_id,
                    e.employee_name,
                    e.job_title,
                    e.manager_id,
                    1 AS chain_step
                FROM employees e
                WHERE e.employee_id = :employee_id

                UNION ALL

                SELECT
                    e.employee_id,
                    e.employee_name,
                    e.job_title,
                    e.manager_id,
                    ac.chain_step + 1
                FROM employees e
                JOIN approval_chain ac ON e.employee_id = ac.manager_id
            )
            SELECT
                ac.chain_step,
                ac.employee_name,
                ac.job_title,
                al.max_amount AS expense_limit
            FROM approval_chain ac
            LEFT JOIN approval_limits al
                   ON al.employee_id = ac.employee_id
                  AND al.limit_type = 'EXPENSE'
            ORDER BY ac.chain_step
        ";

        return DB::select($sql, ['employee_id' => $employeeId]);
    }
}
```

### Laravel Example: Category Tree for API Response

```php
<?php

namespace App\Http\Controllers\Api;

use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
use App\Http\Controllers\Controller;

class CategoryController extends Controller
{
    /**
     * Return the full product category tree as a nested JSON structure.
     *
     * GET /api/categories/tree
     */
    public function tree(): JsonResponse
    {
        $sql = "
            WITH category_tree (
                category_id,
                category_name,
                parent_id,
                slug,
                depth,
                breadcrumb,
                sort_path
            ) AS (
                SELECT
                    c.category_id,
                    c.category_name,
                    c.parent_id,
                    c.slug,
                    0                                         AS depth,
                    CAST(c.category_name AS VARCHAR2(4000))   AS breadcrumb,
                    CAST(LPAD(c.sort_order, 5, '0') AS VARCHAR2(4000)) AS sort_path
                FROM product_categories c
                WHERE c.parent_id IS NULL
                  AND c.is_active = 1

                UNION ALL

                SELECT
                    c.category_id,
                    c.category_name,
                    c.parent_id,
                    c.slug,
                    t.depth + 1,
                    CAST(t.breadcrumb || ' > ' || c.category_name AS VARCHAR2(4000)),
                    CAST(t.sort_path || '-' || LPAD(c.sort_order, 5, '0') AS VARCHAR2(4000))
                FROM product_categories c
                JOIN category_tree t ON c.parent_id = t.category_id
                WHERE c.is_active = 1
            )
            SELECT
                category_id,
                category_name,
                parent_id,
                slug,
                depth,
                breadcrumb
            FROM category_tree
            ORDER BY sort_path
        ";

        $rows = DB::select($sql);

        $tree = $this->buildNestedTree($rows);

        return response()->json([
            'success' => true,
            'data'    => $tree,
        ]);
    }

    /**
     * Return flat category list with depth info — useful for <select> dropdowns.
     *
     * GET /api/categories/flat
     */
    public function flat(): JsonResponse
    {
        $sql = "
            WITH category_tree (
                category_id, category_name, parent_id, slug, depth, sort_path
            ) AS (
                SELECT
                    c.category_id, c.category_name, c.parent_id, c.slug, 0,
                    CAST(LPAD(c.sort_order, 5, '0') AS VARCHAR2(4000))
                FROM product_categories c
                WHERE c.parent_id IS NULL AND c.is_active = 1

                UNION ALL

                SELECT
                    c.category_id, c.category_name, c.parent_id, c.slug, t.depth + 1,
                    CAST(t.sort_path || '-' || LPAD(c.sort_order, 5, '0') AS VARCHAR2(4000))
                FROM product_categories c
                JOIN category_tree t ON c.parent_id = t.category_id
                WHERE c.is_active = 1
            )
            SELECT
                category_id,
                LPAD('—', depth * 2, '—') || ' ' || category_name AS display_name,
                parent_id,
                slug,
                depth
            FROM category_tree
            ORDER BY sort_path
        ";

        $rows = DB::select($sql);

        return response()->json([
            'success' => true,
            'data'    => $rows,
        ]);
    }

    /**
     * Build a nested PHP array tree from flat rows.
     */
    private function buildNestedTree(array $rows): array
    {
        $map = [];
        foreach ($rows as $row) {
            $map[$row->category_id] = [
                'id'       => $row->category_id,
                'name'     => $row->category_name,
                'slug'     => $row->slug,
                'depth'    => $row->depth,
                'children' => [],
            ];
        }

        $tree = [];
        foreach ($map as $id => &$node) {
            // We need the original row's parent_id
        }

        // Simpler rebuild using ordered rows (depth-first order from SQL)
        $stack = [];
        $result = [];

        foreach ($rows as $row) {
            $node = [
                'id'       => $row->category_id,
                'name'     => $row->category_name,
                'slug'     => $row->slug,
                'depth'    => $row->depth,
                'children' => [],
            ];

            if ($row->depth === 0) {
                $result[] = $node;
            }
        }

        return $result;
    }
}
```

### Why Raw SQL Is Better Than Eloquent for Recursive Queries

| Aspect | Eloquent | Raw SQL with DB::select() |
|---|---|---|
| Recursive query support | Not natively supported for Oracle | Full support with all Oracle features |
| Performance | May generate N+1 queries | Single query, all data in one round trip |
| Code readability for complex queries | Complicated chaining | Clear SQL that DBA can review |
| Bind parameter safety | Yes | Yes — always use named bindings |
| Debugging | Hard to inspect generated SQL | SQL is explicit and testable in SQL*Plus |

### Common Laravel + Oracle Recursive CTE Mistakes

```php
// ❌ WRONG: Loading all employees and building hierarchy in PHP
$employees = Employee::all();
$tree = $this->buildTreeInPhp($employees);  // O(n²) PHP loop for large trees

// ✅ RIGHT: Single recursive CTE returns the full tree
$tree = DB::select($recursiveSql);

// ❌ WRONG: String interpolation — SQL injection risk
$sql = "... WHERE manager_id = {$managerId}";

// ✅ RIGHT: Named bind parameters
DB::select($sql, ['manager_id' => $managerId]);

// ❌ WRONG: No index on manager_id column — recursive join is slow
// migration: missing -> $table->index('manager_id');

// ✅ RIGHT: Always index parent/manager columns
// migration: $table->index('manager_id');
//            $table->index('parent_id');

// ❌ WRONG: Running one query per level (N+1 equivalent)
foreach ($topLevelManagers as $manager) {
    $manager->subordinates = DB::select($sql, ['id' => $manager->id]);
}

// ✅ RIGHT: One recursive CTE returns all levels
$allEmployees = DB::select($fullHierarchySql);
```

---

## Practical Real-Life Use Cases

| Domain | Use Case | Recursive CTE Role |
|---|---|---|
| HR / People | Employee hierarchy | Traverse reporting chains |
| HR / People | Organization chart | Build manager-subordinate tree |
| Finance | Approval workflow | Walk chain to approver |
| E-commerce | Product category tree | Build nested menu from flat table |
| CMS | Menu/submenu system | Generate multilevel navigation |
| Sales | Referral network | Count referrals at all depths |
| Manufacturing | Bill of materials | Explode components into sub-components |
| Accounting | Chart of accounts | Summarize accounts at each level |
| File system | Folder/file structure | List all files under a folder |
| Social / Community | Comment thread system | Load nested comment replies |
| Sales | Multi-level sales team | Commission roll-up across levels |
| Banking | Authorization chain | Find all approvers in sequence |
| ERP | Reporting structure | Aggregate P&L across business units |
| IT / Security | Multi-level permissions | Resolve inherited permissions |

---

## Comparison Tables

### Normal CTE vs Recursive CTE

| Aspect | Normal CTE | Recursive CTE |
|---|---|---|
| Self-reference | No | Yes (references its own name in recursive part) |
| Execution | Once | Iteratively until empty result |
| `UNION ALL` required | No | Yes — between anchor and recursive parts |
| Use case | Simplify / reuse subqueries | Hierarchical / tree traversal |
| `SEARCH` clause | No | Optional (depth-first / breadth-first) |
| `CYCLE` clause | No | Optional (cycle detection) |
| Oracle version | 9i+ | 11g R2+ |

### Recursive CTE vs CONNECT BY

| Feature | Recursive CTE | CONNECT BY |
|---|---|---|
| Standard | ANSI SQL:1999 | Oracle proprietary |
| Portability | PostgreSQL, SQL Server, MySQL 8+, DB2 | Oracle only |
| `LEVEL` pseudo-column | Must build manually | Built in |
| `PRIOR` keyword | Not used | Required |
| `SYS_CONNECT_BY_PATH` | Not available (use concatenation) | Available |
| `CONNECT_BY_ROOT` | Not available (use anchor) | Available |
| `CONNECT_BY_ISLEAF` | Not available (use subquery) | Available |
| Complex joins | Easy | Awkward |
| Aggregation in traversal | Full SQL support | Not directly supported |
| Bottom-up traversal | Straightforward | Awkward |
| Cycle detection | Manual or `CYCLE` clause | `NOCYCLE` keyword |
| Verbosity | More lines of SQL | More concise |
| DBA familiarity (Oracle shops) | Growing | Very high |

### Top-Down vs Bottom-Up Traversal

| Aspect | Top-Down | Bottom-Up |
|---|---|---|
| Direction | Root → Leaf | Leaf → Root |
| Use case | Org chart, category tree | Approval chain, path to root |
| Anchor query | Selects root nodes (`manager_id IS NULL`) | Selects starting leaf/employee |
| Recursive join | `child.manager_id = parent.employee_id` | `parent.employee_id = child.manager_id` |
| CONNECT BY equivalent | `CONNECT BY PRIOR employee_id = manager_id` | `CONNECT BY employee_id = PRIOR manager_id` |
| Result order | Parent always before child | Child first, root last |

### Employee Hierarchy vs Product Category Hierarchy

| Aspect | Employee Hierarchy | Product Category Tree |
|---|---|---|
| Root identifier | `manager_id IS NULL` (CEO) | `parent_id IS NULL` (top category) |
| Typical depth | 3–10 levels | 2–6 levels |
| Join column | `manager_id` → `employee_id` | `parent_id` → `category_id` |
| Path use | Approval chains, reporting | Breadcrumbs, slugs, SEO |
| Aggregation | Salary roll-up, headcount | Product count, price range |
| Cycle risk | Medium (user-managed data) | Low (admin-managed) |
| Laravel consumer | HR dashboard, org chart API | Storefront menu, category select |

### Common Mistakes and Fixes

| Mistake | Problem | Fix |
|---|---|---|
| Forgetting column aliases in recursive CTE | Oracle error: columns must be named | Add explicit alias list after CTE name |
| Using `UNION` instead of `UNION ALL` | ORA-32039 error; very slow | Always use `UNION ALL` |
| No stopping condition | Infinite recursion; ORA-600 or memory crash | Ensure parent-child relationship terminates |
| No cycle prevention | Infinite loop on circular data | Add depth limit or `CYCLE` clause |
| Wrong parent-child join column | Returns incorrect rows | Verify join: `child.parent_id = parent.pk` |
| Returning unnecessary columns | Large working set, slow recursion | Select only required columns inside CTE |
| Filtering after recursion | All rows generated before filter | Push `WHERE` conditions into anchor/recursive parts |
| No index on `manager_id` / `parent_id` | Full table scan every iteration | `CREATE INDEX` on join columns |
| Using aggregate functions inside recursion | ORA-32034 error | Aggregate in the final `SELECT`, not inside CTE |
| Simple hierarchy in single table | Overkill complexity | Use `JOIN` or `CONNECT BY` for simple cases |

---

## Common Mistakes

### 1. Forgetting Column Aliases

```sql
-- ❌ WRONG: Oracle requires explicit column list for recursive CTE
WITH emp_hierarchy AS (
    SELECT employee_id, manager_id, 1 FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.employee_id, e.manager_id, h.column3 + 1 -- ORA-00904: ambiguous
    FROM employees e JOIN emp_hierarchy h ON e.manager_id = h.employee_id
)

-- ✅ RIGHT: Name all columns explicitly
WITH emp_hierarchy (employee_id, manager_id, lvl) AS (
    SELECT employee_id, manager_id, 1 FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.employee_id, e.manager_id, h.lvl + 1
    FROM employees e JOIN emp_hierarchy h ON e.manager_id = h.employee_id
)
```

### 2. Using UNION Instead of UNION ALL

```sql
-- ❌ WRONG: Causes ORA-32039
WITH cte (id, parent_id) AS (
    SELECT id, parent_id FROM t WHERE parent_id IS NULL
    UNION                 -- ERROR: ORA-32039
    SELECT c.id, c.parent_id FROM t c JOIN cte p ON c.parent_id = p.id
)

-- ✅ RIGHT
    UNION ALL             -- Must be UNION ALL
```

### 3. No Stopping Condition / No Depth Limit

```sql
-- ❌ RISKY: If data has cycles or unexpected depth, loops forever
WITH cte (id, parent_id, depth) AS (
    SELECT id, parent_id, 0 FROM t WHERE parent_id IS NULL
    UNION ALL
    SELECT c.id, c.parent_id, p.depth + 1 FROM t c JOIN cte p ON c.parent_id = p.id
    -- No depth limit!
)

-- ✅ RIGHT: Add safety depth limit
WHERE p.depth < 100
```

### 4. Filtering Too Late

```sql
-- ❌ SLOW: Generates the entire 10,000-employee tree, then filters to one dept
WITH emp_h (employee_id, department_id, lvl) AS (...)
SELECT * FROM emp_h WHERE department_id = 2;

-- ✅ FASTER: Filter in the anchor so only relevant branches are traversed
WITH emp_h (employee_id, department_id, lvl) AS (
    SELECT employee_id, department_id, 1
    FROM employees
    WHERE manager_id IS NULL AND department_id = 2  -- filter early
    UNION ALL
    ...
)
```

### 5. Aggregate Functions Inside the Recursive Part

```sql
-- ❌ WRONG: ORA-32034 — cannot use aggregate functions in recursive query part
WITH cte (manager_id, total_salary) AS (
    SELECT manager_id, SUM(salary) FROM employees GROUP BY manager_id  -- anchor OK
    UNION ALL
    SELECT e.manager_id, SUM(s.salary)   -- ERROR: SUM in recursive part
    FROM employees e JOIN cte c ON e.manager_id = c.manager_id
    GROUP BY e.manager_id
)

-- ✅ RIGHT: Aggregate in the final SELECT after recursion is complete
WITH cte (employee_id, manager_id) AS (...)
SELECT manager_id, SUM(salary)
FROM cte JOIN employee_salary_history USING (employee_id)
GROUP BY manager_id;
```

---

## Best Practices

1. **Always define explicit column aliases** — list all column names after the CTE name. Oracle requires this for recursive CTEs and it makes code self-documenting.

2. **Keep the anchor query selective** — use `WHERE` conditions in the anchor to start with only the needed root nodes. If you only need one department's tree, filter by department in the anchor.

3. **Always use `UNION ALL`** — never use `UNION` inside a recursive CTE. `UNION` is not allowed in the recursive part and would cause Oracle errors.

4. **Index parent and child key columns** — every recursive join does a lookup by `parent_id` or `manager_id`. Without an index, each recursion level performs a full table scan.

5. **Add a recursion depth column** — always track depth explicitly with a counter (e.g., `lvl` or `depth`). It enables depth-based filtering, indentation, and debugging.

6. **Add maximum depth protection** — for any user-managed hierarchical data, add `WHERE depth < :max_depth` in the recursive part to prevent infinite loops.

7. **Add cycle detection for user-managed data** — either use Oracle's `CYCLE` clause or maintain a visited-ID string. User-managed hierarchies are prone to accidental circular references.

8. **Return only necessary columns inside the CTE** — every extra column in the CTE increases the working set size. Select only what you need inside the recursive definition.

9. **Test with realistic data volume** — a recursive CTE that runs in 50ms on 100 rows may take 30 seconds on 500,000 rows without proper indexes. Always test at scale.

10. **Compare with `CONNECT BY` if Oracle-only** — for simple single-table hierarchies in an Oracle-only environment, `CONNECT BY` is more concise. Use Recursive CTE when you need complex joins, portability, or aggregation.

11. **Use raw SQL in Laravel for complex hierarchy reports** — don't fight Eloquent's limitations. `DB::select()` with named bind parameters is safe, readable, and performant.

12. **Document your recursive CTEs** — add inline comments to the anchor query, recursive part, and stopping conditions. Recursive SQL is harder to read than standard SQL.

13. **Use the `CYCLE` clause for automatic cycle detection** — Oracle 11g R2+ supports it natively. It's cleaner than maintaining a visited-IDs string.

14. **Use `SEARCH DEPTH FIRST` for tree output** — when you need results in a tree-walk order, use the `SEARCH` clause instead of complex `ORDER BY` logic.

15. **Consider materialized views for static hierarchies** — if your category tree or org chart rarely changes, a scheduled materialized view of the recursive CTE result can dramatically improve API response times.

---

## Conclusion

Oracle Recursive CTE (Recursive Subquery Factoring) is a powerful, SQL-standard compliant feature for handling hierarchical and recursive data. It replaced the Oracle-proprietary `CONNECT BY` as the modern, portable, and more flexible approach, especially when:

- Your hierarchy spans multiple joined tables.
- You need to carry computed values (salary roll-up, budget allocation) across levels.
- Your application may run on non-Oracle databases in the future.
- You need clean, readable, testable SQL for complex recursive business logic.

For Laravel developers working with Oracle, Recursive CTE is the right tool for category menus, approval workflows, organization charts, and referral networks. A single well-indexed recursive CTE replaces dozens of PHP loops and N+1 query problems, returning complete hierarchical data in one round trip.

**Key takeaways:**

- Always use `UNION ALL`, never `UNION`.
- Always name CTE columns explicitly.
- Always index `manager_id` / `parent_id` columns.
- Always add a depth limit for user-managed data.
- Use `CYCLE` clause or manual cycle detection for data with circular risk.
- Use raw `DB::select()` in Laravel — never fight Eloquent for recursive queries.
- Prefer Recursive CTE over `CONNECT BY` for all new Oracle development.

---

*This document covers Oracle Database 11g R2 and later. All SQL examples use Oracle-specific syntax. Laravel examples use the `yajra/laravel-oci8` Oracle PDO driver with standard `DB::select()` and named bind parameters.*
