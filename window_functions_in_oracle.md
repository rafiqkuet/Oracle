# Oracle Window Functions / Analytical Functions — Complete Beginner's Guide

> **Audience:** SQL beginners, mid-level developers, and Laravel developers working with Oracle databases.
> **Database:** Oracle SQL (18c / 19c / 21c compatible)

---

## Table of Contents

1. [Introduction](#introduction)
2. [Why Are They Called Analytical Functions?](#why-analytical)
3. [Basic Syntax](#basic-syntax)
4. [Important Clauses](#important-clauses)
5. [Sample Table Setup](#sample-table-setup)
6. [Functions Explained One by One](#functions-explained)
   - [ROW_NUMBER()](#row_number)
   - [RANK()](#rank)
   - [DENSE_RANK()](#dense_rank)
   - [LEAD()](#lead)
   - [LAG()](#lag)
   - [FIRST_VALUE()](#first_value)
   - [LAST_VALUE()](#last_value)
   - [SUM() OVER()](#sum-over)
   - [COUNT() OVER()](#count-over)
   - [AVG() OVER()](#avg-over)
7. [Key Differences Between Functions](#key-differences)
8. [Function Comparison Table](#comparison-table)
9. [Practical Real-Life Examples](#real-life-examples)
10. [Best Practices](#best-practices)
11. [Common Mistakes and How to Avoid Them](#common-mistakes)
12. [Laravel Developer Notes](#laravel-notes)
13. [Conclusion](#conclusion)

---

## 1. Introduction <a name="introduction"></a>

Imagine you want to **rank employees by salary**, or **compare this month's sales to last month's**, or **calculate a running total** — all without losing any rows in your result.

Normal `GROUP BY` collapses your rows into summary rows. **Window Functions** let you perform calculations *across a set of related rows* while **keeping every row in the result intact**.

> **Think of it this way:**
> A regular `SUM()` with `GROUP BY` gives you *one row per group*.
> A `SUM() OVER()` window function gives you *one row per employee*, but each row also shows the group total — both at the same time.

Window functions were introduced in the SQL standard and are **fully supported in Oracle SQL**. They are among the most powerful tools in a data analyst's or backend developer's toolkit.

---

## 2. Why Are They Called Analytical Functions? <a name="why-analytical"></a>

Oracle officially calls these **Analytical Functions** because they are designed to *analyze* data across a logical window (a group of rows) and return a result for **each row individually**.

They are also called **Window Functions** because:

- They define a **"window"** (a subset of rows) relevant to each row being processed.
- The window can slide, expand, or shrink row by row.
- Each row gets its own calculation result based on its own window.

Other database systems (PostgreSQL, SQL Server, MySQL 8+) also support window functions using similar syntax, but Oracle uses the term **Analytical Functions** in its official documentation.

---

## 3. Basic Syntax <a name="basic-syntax"></a>

Every window function follows this general pattern:

```sql
function_name([arguments])
  OVER (
    [PARTITION BY column1, column2, ...]
    [ORDER BY column3 [ASC|DESC]]
    [ROWS BETWEEN ... AND ...]
  )
```

| Part | Required? | Purpose |
|------|-----------|---------|
| `function_name()` | Yes | The function to apply (e.g., `ROW_NUMBER`, `SUM`) |
| `OVER()` | Yes | Marks it as a window/analytical function |
| `PARTITION BY` | No | Divides rows into groups (like `GROUP BY` but keeps all rows) |
| `ORDER BY` inside `OVER()` | Sometimes | Defines the order of rows within each partition |
| `ROWS BETWEEN` | No | Defines the exact frame of rows for the calculation |

---

## 4. Important Clauses <a name="important-clauses"></a>

### `OVER()`

The `OVER()` clause is what turns a regular function into a **window function**. Without it, `SUM()` is just an aggregate. With it, `SUM() OVER()` becomes an analytical function.

```sql
-- Regular aggregate: returns 1 row
SELECT SUM(salary) FROM employees;

-- Window function: returns 1 row PER employee, each showing the total
SELECT employee_name, salary, SUM(salary) OVER() AS total_salary
FROM employees;
```

---

### `PARTITION BY`

`PARTITION BY` divides the result set into **partitions (groups)**. The window function is applied *separately* within each partition.

```sql
-- Calculates total salary per DEPARTMENT, shown on every row
SELECT
  employee_name,
  department,
  salary,
  SUM(salary) OVER (PARTITION BY department) AS dept_total
FROM employees;
```

> **Analogy:** `PARTITION BY` is like `GROUP BY`, but it does **not** collapse your rows. Every employee still appears as their own row.

---

### `ORDER BY` Inside `OVER()`

When you add `ORDER BY` inside the `OVER()` clause, it controls the **order of processing** within each partition. This is essential for ranking functions and running totals.

```sql
-- Rank employees within each department by salary (highest first)
SELECT
  employee_name,
  department,
  salary,
  RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM employees;
```

> ⚠️ **Important:** The `ORDER BY` inside `OVER()` is *separate* from the `ORDER BY` at the end of your `SELECT` query. They serve different purposes.

---

### Window Frame Clauses (`ROWS BETWEEN`)

A **window frame** defines *which rows* around the current row are included in the calculation. This is especially useful for running totals and moving averages.

```sql
-- Syntax
ROWS BETWEEN <start_point> AND <end_point>
```

**Common frame options:**

| Expression | Meaning |
|---|---|
| `UNBOUNDED PRECEDING` | From the very first row of the partition |
| `CURRENT ROW` | The current row being processed |
| `UNBOUNDED FOLLOWING` | Up to the very last row of the partition |
| `N PRECEDING` | N rows before the current row |
| `N FOLLOWING` | N rows after the current row |

**Examples:**

```sql
-- Running total: sum from first row up to current row
SUM(sales) OVER (ORDER BY sale_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)

-- 3-row moving average: current row + 1 before + 1 after
AVG(sales) OVER (ORDER BY sale_date ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING)

-- Total of entire partition (default when no frame is specified)
SUM(sales) OVER (PARTITION BY department)
```

---

## 5. Sample Table Setup <a name="sample-table-setup"></a>

We will use two tables throughout this guide. Run these scripts in your Oracle SQL environment to follow along.

### Create and Populate the `employees` Table

```sql
-- Create employees table
CREATE TABLE employees (
  employee_id   NUMBER PRIMARY KEY,
  employee_name VARCHAR2(50),
  department    VARCHAR2(50),
  salary        NUMBER(10, 2),
  hire_date     DATE
);

-- Insert sample data
INSERT INTO employees VALUES (1,  'Alice',   'Engineering',  90000, DATE '2019-03-15');
INSERT INTO employees VALUES (2,  'Bob',     'Engineering',  85000, DATE '2020-06-01');
INSERT INTO employees VALUES (3,  'Charlie', 'Engineering',  85000, DATE '2021-01-20');
INSERT INTO employees VALUES (4,  'Diana',   'Marketing',    70000, DATE '2018-07-10');
INSERT INTO employees VALUES (5,  'Eve',     'Marketing',    75000, DATE '2020-09-05');
INSERT INTO employees VALUES (6,  'Frank',   'Marketing',    70000, DATE '2022-02-28');
INSERT INTO employees VALUES (7,  'Grace',   'HR',           60000, DATE '2019-11-14');
INSERT INTO employees VALUES (8,  'Henry',   'HR',           62000, DATE '2021-05-30');
INSERT INTO employees VALUES (9,  'Ivy',     'HR',           60000, DATE '2023-01-01');
INSERT INTO employees VALUES (10, 'Jack',    'Engineering',  95000, DATE '2017-08-22');
COMMIT;
```

### Create and Populate the `sales` Table

```sql
-- Create sales table
CREATE TABLE sales (
  sale_id     NUMBER PRIMARY KEY,
  customer_id NUMBER,
  product     VARCHAR2(50),
  sale_amount NUMBER(10, 2),
  sale_date   DATE
);

-- Insert sample data
INSERT INTO sales VALUES (1,  101, 'Laptop',  1200.00, DATE '2024-01-05');
INSERT INTO sales VALUES (2,  102, 'Phone',    800.00, DATE '2024-01-12');
INSERT INTO sales VALUES (3,  101, 'Tablet',   450.00, DATE '2024-02-03');
INSERT INTO sales VALUES (4,  103, 'Laptop',  1200.00, DATE '2024-02-18');
INSERT INTO sales VALUES (5,  102, 'Earbuds',  150.00, DATE '2024-03-07');
INSERT INTO sales VALUES (6,  101, 'Monitor',  350.00, DATE '2024-03-22');
INSERT INTO sales VALUES (7,  103, 'Phone',    800.00, DATE '2024-04-10');
INSERT INTO sales VALUES (8,  104, 'Laptop',  1200.00, DATE '2024-04-15');
INSERT INTO sales VALUES (9,  104, 'Tablet',   450.00, DATE '2024-05-01');
INSERT INTO sales VALUES (10, 102, 'Monitor',  350.00, DATE '2024-05-20');
COMMIT;
```

---

## 6. Functions Explained One by One <a name="functions-explained"></a>

---

### `ROW_NUMBER()` <a name="row_number"></a>

#### Definition

`ROW_NUMBER()` assigns a **unique sequential integer** to each row within a partition, starting from 1. No two rows ever get the same number — even if they have identical values.

#### Syntax

```sql
ROW_NUMBER() OVER (
  [PARTITION BY column]
  ORDER BY column [ASC|DESC]
)
```

#### Practical Use Case

- Picking the **top 1 employee** per department (without `ROWNUM` tricks)
- Assigning unique IDs for pagination
- De-duplicating records by keeping the first occurrence

#### Full Oracle SQL Query

```sql
SELECT
  employee_name,
  department,
  salary,
  ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num
FROM employees
ORDER BY department, salary DESC;
```

#### Expected Output

| EMPLOYEE_NAME | DEPARTMENT   | SALARY   | ROW_NUM |
|---------------|--------------|----------|---------|
| Jack          | Engineering  | 95000.00 | 1       |
| Alice         | Engineering  | 90000.00 | 2       |
| Bob           | Engineering  | 85000.00 | 3       |
| Charlie       | Engineering  | 85000.00 | 4       |
| Henry         | HR           | 62000.00 | 1       |
| Grace         | HR           | 60000.00 | 2       |
| Ivy           | HR           | 60000.00 | 3       |
| Eve           | Marketing    | 75000.00 | 1       |
| Diana         | Marketing    | 70000.00 | 2       |
| Frank         | Marketing    | 70000.00 | 3       |

> **Notice:** Bob and Charlie both earn 85,000, but they get rows 3 and 4 (not both 3). Oracle picks an arbitrary but consistent order when values tie.

#### Step-by-Step Explanation

1. `PARTITION BY department` — Oracle groups employees by department.
2. `ORDER BY salary DESC` — Within each department, rows are sorted highest salary first.
3. `ROW_NUMBER()` — Each row in the partition gets a unique number: 1, 2, 3...

#### Common Beginner Mistake

```sql
-- WRONG: ROW_NUMBER() without ORDER BY is non-deterministic
ROW_NUMBER() OVER (PARTITION BY department)

-- CORRECT: Always specify ORDER BY for predictable numbering
ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC)
```

---

### `RANK()` <a name="rank"></a>

#### Definition

`RANK()` assigns a **rank** to each row within a partition. If two rows tie, they receive the **same rank**, and the next rank is **skipped**. For example: 1, 2, 2, **4** (rank 3 is skipped).

#### Syntax

```sql
RANK() OVER (
  [PARTITION BY column]
  ORDER BY column [ASC|DESC]
)
```

#### Practical Use Case

- Olympic-style rankings (gold, silver, bronze)
- Top-N reports where ties should share the same rank
- Competitive leaderboards

#### Full Oracle SQL Query

```sql
SELECT
  employee_name,
  department,
  salary,
  RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS salary_rank
FROM employees
ORDER BY department, salary DESC;
```

#### Expected Output

| EMPLOYEE_NAME | DEPARTMENT   | SALARY   | SALARY_RANK |
|---------------|--------------|----------|-------------|
| Jack          | Engineering  | 95000.00 | 1           |
| Alice         | Engineering  | 90000.00 | 2           |
| Bob           | Engineering  | 85000.00 | 3           |
| Charlie       | Engineering  | 85000.00 | 3           |
| Henry         | HR           | 62000.00 | 1           |
| Grace         | HR           | 60000.00 | 2           |
| Ivy           | HR           | 60000.00 | 2           |
| Eve           | Marketing    | 75000.00 | 1           |
| Diana         | Marketing    | 70000.00 | 2           |
| Frank         | Marketing    | 70000.00 | 2           |

> **Notice:** Bob and Charlie both have rank 3. There is no rank 4 for Engineering — it jumps to nothing since there are only 4 employees.

#### Common Beginner Mistake

```sql
-- Expecting ranks 1,2,3,4 when there's a tie — but RANK() skips numbers after ties.
-- If you want no gaps, use DENSE_RANK() instead.
```

---

### `DENSE_RANK()` <a name="dense_rank"></a>

#### Definition

`DENSE_RANK()` is like `RANK()`, but it **never skips a number after a tie**. Ties get the same rank, and the next rank is the immediate next integer. For example: 1, 2, 2, **3** (no gap).

#### Syntax

```sql
DENSE_RANK() OVER (
  [PARTITION BY column]
  ORDER BY column [ASC|DESC]
)
```

#### Practical Use Case

- Salary grade reports (Grade 1, Grade 2, Grade 3 — with no gaps)
- Price tier reporting
- Anywhere ties should share a rank without skipping numbers

#### Full Oracle SQL Query

```sql
SELECT
  employee_name,
  department,
  salary,
  DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_salary_rank
FROM employees
ORDER BY department, salary DESC;
```

#### Expected Output

| EMPLOYEE_NAME | DEPARTMENT   | SALARY   | DENSE_SALARY_RANK |
|---------------|--------------|----------|-------------------|
| Jack          | Engineering  | 95000.00 | 1                 |
| Alice         | Engineering  | 90000.00 | 2                 |
| Bob           | Engineering  | 85000.00 | 3                 |
| Charlie       | Engineering  | 85000.00 | 3                 |
| Henry         | HR           | 62000.00 | 1                 |
| Grace         | HR           | 60000.00 | 2                 |
| Ivy           | HR           | 60000.00 | 2                 |
| Eve           | Marketing    | 75000.00 | 1                 |
| Diana         | Marketing    | 70000.00 | 2                 |
| Frank         | Marketing    | 70000.00 | 2                 |

> **Notice:** Bob and Charlie are both rank 3 in Engineering. If there was a 4th salary, it would be rank **4** (not rank 5 as `RANK()` would give).

#### Common Beginner Mistake

Confusing `RANK()` and `DENSE_RANK()`. Always ask: *"Do I want gaps in my ranking after ties?"*
- Yes → Use `RANK()`
- No → Use `DENSE_RANK()`

---

### `LEAD()` <a name="lead"></a>

#### Definition

`LEAD()` lets you **look ahead** to a future row's value — without a self-join. From the current row, it fetches the value of a column from a **following row** within the same partition.

#### Syntax

```sql
LEAD(column, offset, default_value) OVER (
  [PARTITION BY column]
  ORDER BY column
)
```

| Parameter | Description |
|---|---|
| `column` | The column whose value you want from the next row |
| `offset` | How many rows ahead to look (default is 1) |
| `default_value` | What to return if no next row exists (default is NULL) |

#### Practical Use Case

- Comparing **this month's sales** to **next month's sales**
- Finding the **next appointment** for each patient
- Detecting **gaps** in a sequence

#### Full Oracle SQL Query

```sql
SELECT
  customer_id,
  product,
  sale_amount,
  sale_date,
  LEAD(sale_amount, 1, 0) OVER (PARTITION BY customer_id ORDER BY sale_date) AS next_sale_amount,
  LEAD(sale_date,   1)    OVER (PARTITION BY customer_id ORDER BY sale_date) AS next_sale_date
FROM sales
ORDER BY customer_id, sale_date;
```

#### Expected Output

| CUSTOMER_ID | PRODUCT | SALE_AMOUNT | SALE_DATE  | NEXT_SALE_AMOUNT | NEXT_SALE_DATE |
|-------------|---------|-------------|------------|------------------|----------------|
| 101         | Laptop  | 1200.00     | 2024-01-05 | 450.00           | 2024-02-03     |
| 101         | Tablet  | 450.00      | 2024-02-03 | 350.00           | 2024-03-22     |
| 101         | Monitor | 350.00      | 2024-03-22 | **0**            | **NULL**       |
| 102         | Phone   | 800.00      | 2024-01-12 | 150.00           | 2024-03-07     |
| 102         | Earbuds | 150.00      | 2024-03-07 | 350.00           | 2024-05-20     |
| 102         | Monitor | 350.00      | 2024-05-20 | **0**            | **NULL**       |

> **Notice:** The last sale for each customer shows `0` for `NEXT_SALE_AMOUNT` (our default) and `NULL` for `NEXT_SALE_DATE` (no default specified).

#### Common Beginner Mistake

```sql
-- Forgetting to set a default value for the last row — it returns NULL, which
-- can break calculations. Always specify a meaningful default:
LEAD(sale_amount, 1, 0) OVER (...)   -- returns 0 instead of NULL
```

---

### `LAG()` <a name="lag"></a>

#### Definition

`LAG()` is the **opposite of `LEAD()`** — it looks **backward** to a previous row's value within the same partition.

#### Syntax

```sql
LAG(column, offset, default_value) OVER (
  [PARTITION BY column]
  ORDER BY column
)
```

#### Practical Use Case

- Comparing **this month vs last month** sales
- Finding **how long since** the previous event
- Calculating **month-over-month growth**

#### Full Oracle SQL Query

```sql
SELECT
  customer_id,
  product,
  sale_amount,
  sale_date,
  LAG(sale_amount, 1, 0) OVER (PARTITION BY customer_id ORDER BY sale_date) AS prev_sale_amount,
  sale_amount - LAG(sale_amount, 1, 0) OVER (PARTITION BY customer_id ORDER BY sale_date) AS amount_change
FROM sales
ORDER BY customer_id, sale_date;
```

#### Expected Output

| CUSTOMER_ID | PRODUCT | SALE_AMOUNT | SALE_DATE  | PREV_SALE_AMOUNT | AMOUNT_CHANGE |
|-------------|---------|-------------|------------|------------------|---------------|
| 101         | Laptop  | 1200.00     | 2024-01-05 | 0.00             | 1200.00       |
| 101         | Tablet  | 450.00      | 2024-02-03 | 1200.00          | -750.00       |
| 101         | Monitor | 350.00      | 2024-03-22 | 450.00           | -100.00       |
| 102         | Phone   | 800.00      | 2024-01-12 | 0.00             | 800.00        |
| 102         | Earbuds | 150.00      | 2024-03-07 | 800.00           | -650.00       |
| 102         | Monitor | 350.00      | 2024-05-20 | 150.00           | 200.00        |

#### Common Beginner Mistake

```sql
-- Not providing a default for the first row (it would otherwise return NULL
-- and break arithmetic). Use a default of 0 for numeric comparisons:
LAG(sale_amount, 1, 0) OVER (PARTITION BY customer_id ORDER BY sale_date)
```

---

### `FIRST_VALUE()` <a name="first_value"></a>

#### Definition

`FIRST_VALUE()` returns the value of a specified column from the **very first row** in the window frame. It's useful for comparing every row against the "baseline" or "minimum" in its group.

#### Syntax

```sql
FIRST_VALUE(column) OVER (
  [PARTITION BY column]
  ORDER BY column
  [ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW]
)
```

#### Practical Use Case

- Showing the **lowest salary in the department** on every row
- Displaying the **first transaction date** for each customer
- Highlighting the **baseline/minimum value** for comparison

#### Full Oracle SQL Query

```sql
SELECT
  employee_name,
  department,
  salary,
  FIRST_VALUE(salary)      OVER (PARTITION BY department ORDER BY salary DESC
                                 ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS highest_in_dept,
  FIRST_VALUE(employee_name) OVER (PARTITION BY department ORDER BY salary DESC
                                   ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS top_earner
FROM employees
ORDER BY department, salary DESC;
```

#### Expected Output

| EMPLOYEE_NAME | DEPARTMENT   | SALARY   | HIGHEST_IN_DEPT | TOP_EARNER |
|---------------|--------------|----------|-----------------|------------|
| Jack          | Engineering  | 95000.00 | 95000.00        | Jack       |
| Alice         | Engineering  | 90000.00 | 95000.00        | Jack       |
| Bob           | Engineering  | 85000.00 | 95000.00        | Jack       |
| Charlie       | Engineering  | 85000.00 | 95000.00        | Jack       |
| Henry         | HR           | 62000.00 | 62000.00        | Henry      |
| Grace         | HR           | 60000.00 | 62000.00        | Henry      |
| Ivy           | HR           | 60000.00 | 62000.00        | Henry      |

#### Common Beginner Mistake

```sql
-- Without specifying the full frame, FIRST_VALUE() may give unexpected results
-- because the default frame is RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW.
-- Always be explicit:
FIRST_VALUE(salary) OVER (
  PARTITION BY department
  ORDER BY salary DESC
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING  -- see the full partition
)
```

---

### `LAST_VALUE()` <a name="last_value"></a>

#### Definition

`LAST_VALUE()` returns the value from the **last row** in the window frame. This is the counterpart of `FIRST_VALUE()`.

> ⚠️ **Critical note:** By default, Oracle's frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. This means without explicitly extending the frame, `LAST_VALUE()` will return the *current row's value*, not the partition's last value. **Always extend the frame.**

#### Syntax

```sql
LAST_VALUE(column) OVER (
  [PARTITION BY column]
  ORDER BY column
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

#### Practical Use Case

- Showing the **lowest salary** in a department (when sorted DESC, the last row = lowest)
- Finding the **last transaction** for each customer
- Displaying the **most recent event** per group

#### Full Oracle SQL Query

```sql
SELECT
  employee_name,
  department,
  salary,
  LAST_VALUE(salary) OVER (
    PARTITION BY department
    ORDER BY salary DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS lowest_in_dept,
  LAST_VALUE(employee_name) OVER (
    PARTITION BY department
    ORDER BY salary DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS lowest_earner
FROM employees
ORDER BY department, salary DESC;
```

#### Expected Output

| EMPLOYEE_NAME | DEPARTMENT   | SALARY   | LOWEST_IN_DEPT | LOWEST_EARNER |
|---------------|--------------|----------|----------------|---------------|
| Jack          | Engineering  | 95000.00 | 85000.00       | Charlie       |
| Alice         | Engineering  | 90000.00 | 85000.00       | Charlie       |
| Bob           | Engineering  | 85000.00 | 85000.00       | Charlie       |
| Charlie       | Engineering  | 85000.00 | 85000.00       | Charlie       |
| Henry         | HR           | 62000.00 | 60000.00       | Ivy           |
| Grace         | HR           | 60000.00 | 60000.00       | Ivy           |
| Ivy           | HR           | 60000.00 | 60000.00       | Ivy           |

#### Common Beginner Mistake

```sql
-- WRONG: Missing ROWS BETWEEN — returns current row's value, not last row
LAST_VALUE(salary) OVER (PARTITION BY department ORDER BY salary DESC)

-- CORRECT: Extend frame to see the actual last value of the entire partition
LAST_VALUE(salary) OVER (
  PARTITION BY department
  ORDER BY salary DESC
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

---

### `SUM() OVER()` <a name="sum-over"></a>

#### Definition

`SUM() OVER()` calculates a **running total** or a **partition total** — showing the sum alongside every row without collapsing them.

#### Syntax

```sql
SUM(column) OVER (
  [PARTITION BY column]
  [ORDER BY column]
  [ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW]
)
```

#### Practical Use Case

- **Running total** of sales by month
- **Total budget per department** shown on every employee row
- **Cumulative revenue** in a dashboard

#### Full Oracle SQL Query

```sql
SELECT
  employee_name,
  department,
  salary,
  -- Total salary for the whole company
  SUM(salary) OVER ()                                                    AS company_total,
  -- Total salary per department
  SUM(salary) OVER (PARTITION BY department)                             AS dept_total,
  -- Running cumulative salary within each department (sorted by hire date)
  SUM(salary) OVER (PARTITION BY department ORDER BY hire_date
                    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)    AS running_dept_total
FROM employees
ORDER BY department, hire_date;
```

#### Expected Output (Engineering only, for brevity)

| EMPLOYEE_NAME | DEPARTMENT  | SALARY   | COMPANY_TOTAL | DEPT_TOTAL | RUNNING_DEPT_TOTAL |
|---------------|-------------|----------|---------------|------------|--------------------|
| Jack          | Engineering | 95000.00 | 752000.00     | 355000.00  | 95000.00           |
| Alice         | Engineering | 90000.00 | 752000.00     | 355000.00  | 185000.00          |
| Bob           | Engineering | 85000.00 | 752000.00     | 355000.00  | 270000.00          |
| Charlie       | Engineering | 85000.00 | 752000.00     | 355000.00  | 355000.00          |

#### Common Beginner Mistake

```sql
-- Adding ORDER BY without a frame turns SUM() into a running total automatically.
-- This is usually what you want, but beginners are surprised by it:
SUM(salary) OVER (PARTITION BY department ORDER BY hire_date)
-- The above IS a running total (RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW by default).

-- If you want the FULL department total, remove ORDER BY:
SUM(salary) OVER (PARTITION BY department)
```

---

### `COUNT() OVER()` <a name="count-over"></a>

#### Definition

`COUNT() OVER()` counts rows within a window, **showing the count on every row** without collapsing the result. Very useful for showing "how many are in my group" while keeping individual row details.

#### Syntax

```sql
COUNT(*) OVER (
  [PARTITION BY column]
  [ORDER BY column]
)
```

#### Practical Use Case

- "How many employees are in my department?" — shown on every row
- Total number of transactions per customer
- Pagination metadata (total records in result set)

#### Full Oracle SQL Query

```sql
SELECT
  employee_name,
  department,
  salary,
  COUNT(*) OVER ()                           AS total_employees,
  COUNT(*) OVER (PARTITION BY department)    AS dept_headcount,
  COUNT(*) OVER (PARTITION BY department
                 ORDER BY hire_date
                 ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cumulative_hires
FROM employees
ORDER BY department, hire_date;
```

#### Expected Output (Engineering only, for brevity)

| EMPLOYEE_NAME | DEPARTMENT  | SALARY   | TOTAL_EMPLOYEES | DEPT_HEADCOUNT | CUMULATIVE_HIRES |
|---------------|-------------|----------|-----------------|----------------|------------------|
| Jack          | Engineering | 95000.00 | 10              | 4              | 1                |
| Alice         | Engineering | 90000.00 | 10              | 4              | 2                |
| Bob           | Engineering | 85000.00 | 10              | 4              | 3                |
| Charlie       | Engineering | 85000.00 | 10              | 4              | 4                |

#### Common Beginner Mistake

```sql
-- COUNT(column) vs COUNT(*) — if the column has NULLs, COUNT(column) skips them.
-- For headcounts, always use COUNT(*):
COUNT(*) OVER (PARTITION BY department)       -- counts all rows (correct for headcount)
COUNT(salary) OVER (PARTITION BY department)  -- skips NULLs (may undercount)
```

---

### `AVG() OVER()` <a name="avg-over"></a>

#### Definition

`AVG() OVER()` computes an **average within a window** and attaches the result to every row — without reducing the number of rows returned.

#### Syntax

```sql
AVG(column) OVER (
  [PARTITION BY column]
  [ORDER BY column]
  [ROWS BETWEEN ...]
)
```

#### Practical Use Case

- Show **average department salary** next to each employee
- Calculate a **3-month moving average** for sales
- Flag employees who earn **below department average**

#### Full Oracle SQL Query

```sql
SELECT
  employee_name,
  department,
  salary,
  ROUND(AVG(salary) OVER (PARTITION BY department), 2)   AS dept_avg_salary,
  salary - ROUND(AVG(salary) OVER (PARTITION BY department), 2) AS diff_from_avg,
  CASE
    WHEN salary > AVG(salary) OVER (PARTITION BY department) THEN 'Above Average'
    WHEN salary < AVG(salary) OVER (PARTITION BY department) THEN 'Below Average'
    ELSE 'At Average'
  END AS performance_tag
FROM employees
ORDER BY department, salary DESC;
```

#### Expected Output

| EMPLOYEE_NAME | DEPARTMENT   | SALARY   | DEPT_AVG_SALARY | DIFF_FROM_AVG | PERFORMANCE_TAG |
|---------------|--------------|----------|-----------------|---------------|-----------------|
| Jack          | Engineering  | 95000.00 | 88750.00        | 6250.00       | Above Average   |
| Alice         | Engineering  | 90000.00 | 88750.00        | 1250.00       | Above Average   |
| Bob           | Engineering  | 85000.00 | 88750.00        | -3750.00      | Below Average   |
| Charlie       | Engineering  | 85000.00 | 88750.00        | -3750.00      | Below Average   |
| Eve           | Marketing    | 75000.00 | 71666.67        | 3333.33       | Above Average   |
| Diana         | Marketing    | 70000.00 | 71666.67        | -1666.67      | Below Average   |
| Frank         | Marketing    | 70000.00 | 71666.67        | -1666.67      | Below Average   |

#### Common Beginner Mistake

```sql
-- Trying to filter rows using a window function in WHERE clause — this DOES NOT work:
-- WRONG:
SELECT * FROM employees
WHERE AVG(salary) OVER (PARTITION BY department) > 70000;  -- ERROR!

-- CORRECT: Wrap in a subquery or use a CTE:
SELECT * FROM (
  SELECT
    employee_name,
    department,
    salary,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg
  FROM employees
)
WHERE dept_avg > 70000;
```

---

## 7. Key Differences Between Functions <a name="key-differences"></a>

### `ROW_NUMBER()` vs `RANK()` vs `DENSE_RANK()`

Consider three employees all earning 85,000 and one earning 90,000:

```sql
SELECT
  employee_name, salary,
  ROW_NUMBER()  OVER (ORDER BY salary DESC) AS row_num,
  RANK()        OVER (ORDER BY salary DESC) AS rnk,
  DENSE_RANK()  OVER (ORDER BY salary DESC) AS dense_rnk
FROM employees
WHERE department = 'Engineering'
ORDER BY salary DESC;
```

| EMPLOYEE_NAME | SALARY   | ROW_NUM | RNK | DENSE_RNK |
|---------------|----------|---------|-----|-----------|
| Jack          | 95000.00 | 1       | 1   | 1         |
| Alice         | 90000.00 | 2       | 2   | 2         |
| Bob           | 85000.00 | 3       | 3   | 3         |
| Charlie       | 85000.00 | 4       | 3   | 3         |

> **Summary:**
> - `ROW_NUMBER()` → Always unique. Never repeats. No ties possible.
> - `RANK()` → Ties share the same rank. **Next rank is skipped** (gap after tie).
> - `DENSE_RANK()` → Ties share the same rank. **No gaps** in ranking sequence.

---

### `LEAD()` vs `LAG()`

| Feature | `LEAD()` | `LAG()` |
|---|---|---|
| Direction | Looks **forward** (next rows) | Looks **backward** (previous rows) |
| Use case | Compare to future value | Compare to past value |
| First row | Has a value to lead to | Returns `NULL` (or default) |
| Last row | Returns `NULL` (or default) | Has a value to lag from |
| Example | Next month's sales | Last month's sales |

---

### Aggregate with `GROUP BY` vs Aggregate with `OVER()`

```sql
-- GROUP BY: collapses rows — you lose individual employee detail
SELECT department, AVG(salary) AS avg_salary
FROM employees
GROUP BY department;
-- Returns 3 rows (one per department)

-- OVER(): keeps all rows — each row retains its detail PLUS the aggregate
SELECT employee_name, department, salary,
       AVG(salary) OVER (PARTITION BY department) AS avg_salary
FROM employees;
-- Returns 10 rows (one per employee), each with their department average
```

| Feature | `GROUP BY` Aggregate | `OVER()` Window Function |
|---|---|---|
| Rows returned | One per group | One per original row |
| Individual row access | Lost | Preserved |
| Can mix detail + summary | No | Yes |
| Use in `WHERE` clause | Can use `HAVING` | Must wrap in subquery |
| Performance | Generally faster | Slightly more overhead |

---

## 8. Function Comparison Table <a name="comparison-table"></a>

| Function | Type | Returns | Requires `ORDER BY`? | Handles Ties | Common Use Case |
|---|---|---|---|---|---|
| `ROW_NUMBER()` | Ranking | Unique sequential int | Yes | No (always unique) | Pagination, deduplication |
| `RANK()` | Ranking | Rank with gaps | Yes | Yes (same rank + gap) | Olympic rankings, leaderboards |
| `DENSE_RANK()` | Ranking | Rank without gaps | Yes | Yes (same rank, no gap) | Grade tiers, salary bands |
| `LEAD()` | Navigation | Next row's value | Yes | N/A | Compare to future |
| `LAG()` | Navigation | Previous row's value | Yes | N/A | Compare to past |
| `FIRST_VALUE()` | Navigation | First row in frame | Yes | N/A | Baseline comparison |
| `LAST_VALUE()` | Navigation | Last row in frame | Yes | N/A | Most recent in group |
| `SUM() OVER()` | Aggregate | Running / group sum | Optional | N/A | Running totals, budgets |
| `COUNT() OVER()` | Aggregate | Row count in window | Optional | N/A | Headcount per group |
| `AVG() OVER()` | Aggregate | Average in window | Optional | N/A | Dept avg on every row |

---

## 9. Practical Real-Life Examples <a name="real-life-examples"></a>

### Example 1: Rank Employees by Salary (Company-Wide Leaderboard)

```sql
SELECT
  employee_name,
  department,
  salary,
  DENSE_RANK() OVER (ORDER BY salary DESC)                        AS company_rank,
  DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM employees
ORDER BY company_rank;
```

| EMPLOYEE_NAME | DEPARTMENT  | SALARY   | COMPANY_RANK | DEPT_RANK |
|---------------|-------------|----------|--------------|-----------|
| Jack          | Engineering | 95000.00 | 1            | 1         |
| Alice         | Engineering | 90000.00 | 2            | 2         |
| Bob           | Engineering | 85000.00 | 3            | 3         |
| Charlie       | Engineering | 85000.00 | 3            | 3         |
| Eve           | Marketing   | 75000.00 | 4            | 1         |
| Diana         | Marketing   | 70000.00 | 5            | 2         |
| Frank         | Marketing   | 70000.00 | 5            | 2         |
| Henry         | HR          | 62000.00 | 6            | 1         |
| Grace         | HR          | 60000.00 | 7            | 2         |
| Ivy           | HR          | 60000.00 | 7            | 2         |

---

### Example 2: Find the Top-Selling Product Per Customer

```sql
SELECT *
FROM (
  SELECT
    customer_id,
    product,
    SUM(sale_amount)                                           AS total_spent,
    RANK() OVER (PARTITION BY customer_id ORDER BY SUM(sale_amount) DESC) AS product_rank
  FROM sales
  GROUP BY customer_id, product
)
WHERE product_rank = 1
ORDER BY customer_id;
```

| CUSTOMER_ID | PRODUCT | TOTAL_SPENT | PRODUCT_RANK |
|-------------|---------|-------------|--------------|
| 101         | Laptop  | 1200.00     | 1            |
| 102         | Phone   | 800.00      | 1            |
| 103         | Laptop  | 1200.00     | 1            |
| 104         | Laptop  | 1200.00     | 1            |

---

### Example 3: Compare Current Month Sales with Previous Month (MoM)

```sql
SELECT
  TO_CHAR(sale_date, 'YYYY-MM')                                  AS sale_month,
  SUM(sale_amount)                                               AS monthly_sales,
  LAG(SUM(sale_amount)) OVER (ORDER BY TO_CHAR(sale_date, 'YYYY-MM')) AS prev_month_sales,
  SUM(sale_amount) - LAG(SUM(sale_amount)) OVER (ORDER BY TO_CHAR(sale_date, 'YYYY-MM')) AS mom_change,
  ROUND(
    (SUM(sale_amount) - LAG(SUM(sale_amount)) OVER (ORDER BY TO_CHAR(sale_date, 'YYYY-MM')))
    / LAG(SUM(sale_amount)) OVER (ORDER BY TO_CHAR(sale_date, 'YYYY-MM')) * 100, 2
  ) AS mom_growth_pct
FROM sales
GROUP BY TO_CHAR(sale_date, 'YYYY-MM')
ORDER BY sale_month;
```

| SALE_MONTH | MONTHLY_SALES | PREV_MONTH_SALES | MOM_CHANGE | MOM_GROWTH_PCT |
|------------|---------------|------------------|------------|----------------|
| 2024-01    | 2000.00       | NULL             | NULL       | NULL           |
| 2024-02    | 1650.00       | 2000.00          | -350.00    | -17.50         |
| 2024-03    | 500.00        | 1650.00          | -1150.00   | -69.70         |
| 2024-04    | 2000.00       | 500.00           | 1500.00    | 300.00         |
| 2024-05    | 800.00        | 2000.00          | -1200.00   | -60.00         |

---

### Example 4: Calculating Running Total of Sales

```sql
SELECT
  sale_id,
  customer_id,
  product,
  sale_amount,
  sale_date,
  SUM(sale_amount) OVER (
    ORDER BY sale_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS running_total
FROM sales
ORDER BY sale_date;
```

| SALE_ID | CUSTOMER_ID | PRODUCT | SALE_AMOUNT | SALE_DATE  | RUNNING_TOTAL |
|---------|-------------|---------|-------------|------------|---------------|
| 1       | 101         | Laptop  | 1200.00     | 2024-01-05 | 1200.00       |
| 2       | 102         | Phone   | 800.00      | 2024-01-12 | 2000.00       |
| 3       | 101         | Tablet  | 450.00      | 2024-02-03 | 2450.00       |
| 4       | 103         | Laptop  | 1200.00     | 2024-02-18 | 3650.00       |
| 5       | 102         | Earbuds | 150.00      | 2024-03-07 | 3800.00       |
| 6       | 101         | Monitor | 350.00      | 2024-03-22 | 4150.00       |

---

### Example 5: Average Salary by Department Without Losing Employee Rows

```sql
SELECT
  employee_name,
  department,
  salary,
  ROUND(AVG(salary) OVER (PARTITION BY department), 2) AS dept_avg,
  ROUND((salary / AVG(salary) OVER (PARTITION BY department)) * 100, 1) AS pct_of_dept_avg
FROM employees
ORDER BY department, salary DESC;
```

| EMPLOYEE_NAME | DEPARTMENT   | SALARY   | DEPT_AVG | PCT_OF_DEPT_AVG |
|---------------|--------------|----------|----------|-----------------|
| Jack          | Engineering  | 95000.00 | 88750.00 | 107.0%          |
| Alice         | Engineering  | 90000.00 | 88750.00 | 101.4%          |
| Bob           | Engineering  | 85000.00 | 88750.00 | 95.8%           |
| Charlie       | Engineering  | 85000.00 | 88750.00 | 95.8%           |

---

### Example 6: First and Last Transaction for Each Customer

```sql
SELECT DISTINCT
  customer_id,
  FIRST_VALUE(product) OVER (
    PARTITION BY customer_id ORDER BY sale_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS first_product,
  FIRST_VALUE(sale_date) OVER (
    PARTITION BY customer_id ORDER BY sale_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS first_purchase_date,
  LAST_VALUE(product) OVER (
    PARTITION BY customer_id ORDER BY sale_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS last_product,
  LAST_VALUE(sale_date) OVER (
    PARTITION BY customer_id ORDER BY sale_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS last_purchase_date,
  COUNT(*) OVER (PARTITION BY customer_id) AS total_purchases
FROM sales
ORDER BY customer_id;
```

| CUSTOMER_ID | FIRST_PRODUCT | FIRST_PURCHASE_DATE | LAST_PRODUCT | LAST_PURCHASE_DATE | TOTAL_PURCHASES |
|-------------|---------------|---------------------|--------------|---------------------|-----------------|
| 101         | Laptop        | 2024-01-05          | Monitor      | 2024-03-22          | 3               |
| 102         | Phone         | 2024-01-12          | Monitor      | 2024-05-20          | 3               |
| 103         | Laptop        | 2024-02-18          | Phone        | 2024-04-10          | 2               |
| 104         | Laptop        | 2024-04-15          | Tablet       | 2024-05-01          | 2               |

---

### Example 7: Complete Dashboard / Report Query

This query is typical of what a Laravel admin panel backend might power:

```sql
SELECT
  e.employee_name,
  e.department,
  e.salary,
  e.hire_date,
  -- Rankings
  DENSE_RANK() OVER (ORDER BY e.salary DESC)                         AS company_salary_rank,
  DENSE_RANK() OVER (PARTITION BY e.department ORDER BY e.salary DESC) AS dept_salary_rank,
  ROW_NUMBER() OVER (PARTITION BY e.department ORDER BY e.hire_date)  AS hire_order_in_dept,
  -- Aggregates alongside individual rows
  ROUND(AVG(e.salary) OVER (PARTITION BY e.department), 2)           AS dept_avg_salary,
  SUM(e.salary) OVER (PARTITION BY e.department)                     AS dept_salary_budget,
  SUM(e.salary) OVER ()                                              AS total_payroll,
  COUNT(*) OVER (PARTITION BY e.department)                          AS dept_headcount,
  -- Performance tagging
  CASE
    WHEN e.salary > AVG(e.salary) OVER (PARTITION BY e.department)
    THEN 'Above Average'
    ELSE 'At or Below Average'
  END AS salary_performance
FROM employees e
ORDER BY e.department, e.salary DESC;
```

---

## 10. Best Practices <a name="best-practices"></a>

### 1. Always Use Explicit `ROWS BETWEEN` for `LAST_VALUE()`

`LAST_VALUE()` without a frame clause is almost always wrong. Specify `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` to capture the true last value.

### 2. Use CTEs for Readability

Nesting window functions inside subqueries can get messy. Use Common Table Expressions (CTEs):

```sql
WITH ranked_employees AS (
  SELECT
    employee_name,
    department,
    salary,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
  FROM employees
)
SELECT * FROM ranked_employees WHERE dept_rank = 1;
```

### 3. Do Not Use Window Functions in `WHERE` — Use Subqueries or CTEs

Window functions are evaluated **after** `WHERE`. You cannot filter on them in the `WHERE` clause directly.

```sql
-- WRONG:
SELECT * FROM employees WHERE RANK() OVER (ORDER BY salary DESC) <= 3;

-- CORRECT:
SELECT * FROM (
  SELECT employee_name, salary,
         RANK() OVER (ORDER BY salary DESC) AS rnk
  FROM employees
) WHERE rnk <= 3;
```

### 4. Choose the Right Ranking Function

- Pagination → `ROW_NUMBER()`
- Olympic-style / competitive → `RANK()`
- Salary grades / tiers → `DENSE_RANK()`

### 5. Always Provide a Default Value for `LEAD()` and `LAG()`

Null values from the first/last row can break calculations. Always use the third parameter:

```sql
LAG(salary, 1, 0)   OVER (...)   -- 0 instead of NULL for numeric
LEAD(status, 1, 'N/A') OVER (...) -- 'N/A' instead of NULL for strings
```

### 6. Minimize Redundant `OVER()` Clauses

If you use the same window definition multiple times, define it once using the `WINDOW` alias (supported in newer Oracle versions) or use a CTE.

### 7. Index the Columns Used in `PARTITION BY` and `ORDER BY`

Window functions can be expensive on large tables. Proper indexing on partition and order columns dramatically improves performance.

---

## 11. Common Mistakes and How to Avoid Them <a name="common-mistakes"></a>

| # | Mistake | Problem | Fix |
|---|---------|---------|-----|
| 1 | Using a window function in `WHERE` | Oracle evaluates window functions after `WHERE` — error or wrong result | Wrap in a subquery or CTE |
| 2 | `LAST_VALUE()` without frame | Returns current row's value, not the partition's last | Add `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` |
| 3 | `ROW_NUMBER()` without `ORDER BY` | Non-deterministic — rows get numbers in unpredictable order | Always specify `ORDER BY` |
| 4 | Confusing `RANK()` and `DENSE_RANK()` | Gaps in ranks when using `RANK()` unexpectedly | Know the rule: `RANK()` has gaps, `DENSE_RANK()` does not |
| 5 | `LEAD()`/`LAG()` returning `NULL` | Breaks arithmetic on first/last rows | Always provide a default: `LAG(col, 1, 0)` |
| 6 | Mixing window `ORDER BY` with query `ORDER BY` | Confusing the two — they are completely independent | Remember: `ORDER BY` inside `OVER()` controls the window; the outer `ORDER BY` controls display order |
| 7 | Expecting `AVG() OVER()` to behave like `GROUP BY` | Window functions keep all rows; `GROUP BY` collapses them | Use window functions when you need detail + aggregate together |
| 8 | Using `SUM() OVER (ORDER BY ...)` and expecting a full total | Adding `ORDER BY` to `SUM() OVER()` creates a running total, not a full sum | Remove `ORDER BY` for full partition total: `SUM(col) OVER (PARTITION BY dept)` |
| 9 | Not aliasing window function columns | Hard to reference or join later | Always alias: `SUM(salary) OVER (...) AS dept_total` |
| 10 | Applying window functions to already-aggregated data without CTE | Complex, error-prone SQL | Break into CTE stages: aggregate first, then apply window function |

---

## 12. Laravel Developer Notes <a name="laravel-notes"></a>

### When to Use Window Functions in Laravel Projects

Window functions are ideal in Laravel for:

| Laravel Feature | Window Function Use Case |
|---|---|
| **Admin dashboards** | Show each employee's salary alongside department average |
| **Reports / exports** | Ranked lists, running totals for Excel/PDF exports |
| **API responses** | Return rank + individual data in one query (fewer round trips) |
| **Pagination helpers** | Use `ROW_NUMBER()` for Oracle-compatible pagination |
| **Analytics panels** | Month-over-month comparisons with `LAG()` |
| **Customer portals** | First/last purchase dates using `FIRST_VALUE()` / `LAST_VALUE()` |

---

### Why Window Functions Beat Multiple Laravel Queries

Instead of making multiple Eloquent/DB queries to get both individual data and group summaries (N+1 problem), one window function query returns everything at once.

```php
// BAD: N+1 approach — one query per department
$departments = DB::select("SELECT DISTINCT department FROM employees");
foreach ($departments as $dept) {
    $avg = DB::select("SELECT AVG(salary) FROM employees WHERE department = ?", [$dept->department]);
    // ...
}

// GOOD: One query using window function — all data at once
$employees = DB::select("
    SELECT
        employee_name,
        department,
        salary,
        ROUND(AVG(salary) OVER (PARTITION BY department), 2) AS dept_avg_salary,
        DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
    FROM employees
    ORDER BY department, salary DESC
");
```

---

### Example: Salary Ranking API in Laravel

```php
<?php

namespace App\Http\Controllers\Api;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;

class EmployeeReportController extends Controller
{
    /**
     * GET /api/employees/salary-report
     * Returns all employees with department ranking and averages.
     */
    public function salaryReport()
    {
        $results = DB::select("
            SELECT
                employee_name,
                department,
                salary,
                hire_date,
                DENSE_RANK() OVER (ORDER BY salary DESC)                          AS company_rank,
                DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC)  AS dept_rank,
                ROUND(AVG(salary) OVER (PARTITION BY department), 2)              AS dept_avg_salary,
                SUM(salary) OVER (PARTITION BY department)                        AS dept_total_salary,
                COUNT(*) OVER (PARTITION BY department)                           AS dept_headcount,
                CASE
                    WHEN salary > AVG(salary) OVER (PARTITION BY department)
                    THEN 'Above Average'
                    ELSE 'At or Below Average'
                END AS salary_tag
            FROM employees
            ORDER BY company_rank, department
        ");

        return response()->json([
            'status'  => 'success',
            'count'   => count($results),
            'data'    => $results,
        ]);
    }

    /**
     * GET /api/sales/monthly-comparison
     * Returns monthly sales with month-over-month growth.
     */
    public function monthlySalesComparison()
    {
        $results = DB::select("
            SELECT
                sale_month,
                monthly_sales,
                prev_month_sales,
                monthly_sales - prev_month_sales AS mom_change,
                CASE
                    WHEN prev_month_sales IS NULL THEN NULL
                    ELSE ROUND(
                        (monthly_sales - prev_month_sales) / prev_month_sales * 100, 2
                    )
                END AS mom_growth_pct
            FROM (
                SELECT
                    TO_CHAR(sale_date, 'YYYY-MM') AS sale_month,
                    SUM(sale_amount)              AS monthly_sales,
                    LAG(SUM(sale_amount)) OVER (
                        ORDER BY TO_CHAR(sale_date, 'YYYY-MM')
                    )                             AS prev_month_sales
                FROM sales
                GROUP BY TO_CHAR(sale_date, 'YYYY-MM')
            )
            ORDER BY sale_month
        ");

        return response()->json([
            'status' => 'success',
            'data'   => $results,
        ]);
    }

    /**
     * GET /api/customers/purchase-summary
     * Returns first and last purchase details per customer.
     */
    public function customerPurchaseSummary()
    {
        $results = DB::select("
            SELECT DISTINCT
                customer_id,
                FIRST_VALUE(product) OVER (
                    PARTITION BY customer_id ORDER BY sale_date
                    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
                ) AS first_product,
                FIRST_VALUE(sale_date) OVER (
                    PARTITION BY customer_id ORDER BY sale_date
                    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
                ) AS first_purchase_date,
                LAST_VALUE(product) OVER (
                    PARTITION BY customer_id ORDER BY sale_date
                    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
                ) AS last_product,
                LAST_VALUE(sale_date) OVER (
                    PARTITION BY customer_id ORDER BY sale_date
                    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
                ) AS last_purchase_date,
                COUNT(*) OVER (PARTITION BY customer_id)         AS total_purchases,
                SUM(sale_amount) OVER (PARTITION BY customer_id) AS lifetime_value
            FROM sales
            ORDER BY customer_id
        ");

        return response()->json([
            'status' => 'success',
            'data'   => $results,
        ]);
    }
}
```

---

### Oracle-Specific Pagination Using `ROW_NUMBER()` in Laravel

Oracle does not support `LIMIT`/`OFFSET`. Use `ROW_NUMBER()` instead:

```php
<?php

public function paginatedEmployees(Request $request)
{
    $page    = $request->input('page', 1);
    $perPage = $request->input('per_page', 5);
    $offset  = ($page - 1) * $perPage;
    $limit   = $offset + $perPage;

    $results = DB::select("
        SELECT *
        FROM (
            SELECT
                employee_name,
                department,
                salary,
                ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn
            FROM employees
        )
        WHERE rn > :offset AND rn <= :limit
    ", [
        'offset' => $offset,
        'limit'  => $limit,
    ]);

    $total = DB::selectOne("SELECT COUNT(*) AS cnt FROM employees")->cnt;

    return response()->json([
        'status'       => 'success',
        'current_page' => $page,
        'per_page'     => $perPage,
        'total'        => $total,
        'data'         => $results,
    ]);
}
```

---

## 13. Conclusion <a name="conclusion"></a>

Oracle Window Functions (Analytical Functions) are one of the most powerful features of SQL. They let you:

- **Rank and number rows** without subqueries or complex joins
- **Compare rows to their neighbors** using `LEAD()` and `LAG()`
- **Calculate running totals, moving averages, and group sums** while keeping every row intact
- **Identify the first and last values** in any group with `FIRST_VALUE()` and `LAST_VALUE()`

### Key Takeaways

| Concept | Remember This |
|---|---|
| `OVER()` | Turns any function into a window function |
| `PARTITION BY` | Groups rows (like `GROUP BY`) but keeps all rows visible |
| `ORDER BY` in `OVER()` | Controls row order within the window — not the query output |
| `ROWS BETWEEN` | Defines the exact frame of rows included in the calculation |
| `ROW_NUMBER` vs `RANK` vs `DENSE_RANK` | Unique / Gaps-on-tie / No-gaps-on-tie |
| `LEAD` / `LAG` | Look ahead / look behind within a partition |
| Window functions in `WHERE` | Not allowed — wrap in a subquery or CTE |

For Laravel developers working with Oracle, mastering window functions means:
- Fewer database round trips
- Simpler application code
- Faster, more efficient reporting queries
- Cleaner APIs that return enriched data in a single call

> **Start with `ROW_NUMBER()` for pagination and `AVG() OVER()` for department comparisons — you'll quickly see why window functions are indispensable.**

---

*Document prepared for Oracle SQL 18c/19c/21c. Window function syntax is consistent across these versions.*
*Laravel examples use `DB::select()` with raw Oracle SQL via the `yajra/laravel-oci8` or standard PDO Oracle driver.*
