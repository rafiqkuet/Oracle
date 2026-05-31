# Oracle SQL — Common Table Expression (CTE) Complete Guide

> A beginner-friendly, practical reference for developers working with Oracle SQL and CTEs.

---

## Table of Contents

1. [What is a CTE in Oracle?](#1-what-is-a-cte-in-oracle)
2. [Why CTE is Called the `WITH` Clause](#2-why-cte-is-called-the-with-clause)
3. [Basic Syntax of CTE in Oracle SQL](#3-basic-syntax-of-cte-in-oracle-sql)
4. [Advantages of Using CTE](#4-advantages-of-using-cte)
5. [CTE vs Subquery vs View](#5-cte-vs-subquery-vs-view)
6. [Full Working Oracle SQL Example](#6-full-working-oracle-sql-example)
7. [Practical Real-Life Examples](#7-practical-real-life-examples)
8. [How CTE Improves Query Readability](#8-how-cte-improves-query-readability)
9. [When to Use CTE — and When Not To](#9-when-to-use-cte--and-when-not-to)
10. [Common Mistakes Beginners Make](#10-common-mistakes-beginners-make)
11. [Best Practices for Real Projects](#11-best-practices-for-real-projects)
12. [Laravel + Oracle CTE Examples](#12-laravel--oracle-cte-examples)

---

## 1. What is a CTE in Oracle?

A **Common Table Expression (CTE)** is a **named, temporary result set** that you define at the beginning of a SQL query and can reference inside that same query.

Think of it like giving a nickname to a sub-query so you can use it by name instead of copy-pasting the same code over and over.

### Simple analogy

Imagine you are writing a long essay. Instead of writing "the Department of Human Resources" every time, you say:

> *"Let's call it HR from now on."*

A CTE does the same thing for your SQL queries — it lets you name a piece of logic once and reuse it cleanly.

### Key facts about CTE in Oracle

- Supported since **Oracle 9i** (basic CTE) and **Oracle 11g R2** (recursive CTE).
- Exists **only for the duration of the single query** — it is not stored in the database.
- Defined using the **`WITH`** keyword.
- Can reference **multiple CTEs** in one query.
- Makes complex queries **easier to read, write, and maintain**.

---

## 2. Why CTE is Called the `WITH` Clause

Oracle SQL uses the keyword **`WITH`** to start a CTE. This is why developers often call it the **"WITH clause"** — it is simply named after the SQL keyword used to write it.

```sql
WITH cte_name AS (
    -- your query goes here
)
SELECT * FROM cte_name;
```

The `WITH` keyword tells Oracle:

> *"Before running the main query, first prepare this named result set."*

Both names refer to the same thing:

| Term | Meaning |
|------|---------|
| CTE | Common Table Expression — the formal SQL standard term |
| WITH Clause | What developers call it based on the keyword used |
| Subquery Factoring | Oracle's own official name for CTEs in its documentation |

---

## 3. Basic Syntax of CTE in Oracle SQL

### Single CTE

```sql
WITH cte_name AS (
    SELECT column1, column2
    FROM table_name
    WHERE condition
)
SELECT *
FROM cte_name;
```

### Multiple CTEs in one query

```sql
WITH
    first_cte AS (
        SELECT column1 FROM table_a
    ),
    second_cte AS (
        SELECT column2 FROM table_b
    )
SELECT f.column1, s.column2
FROM first_cte f
JOIN second_cte s ON f.column1 = s.column2;
```

### Syntax rules to remember

- The `WITH` keyword comes **before** the main `SELECT`.
- Each CTE is wrapped in **parentheses**.
- Multiple CTEs are separated by **commas**.
- A **semicolon** ends the entire statement — not the CTE itself.
- The main query (`SELECT`, `INSERT`, `UPDATE`, `DELETE`) comes **after** all CTEs.

---

## 4. Advantages of Using CTE

| Advantage | Description |
|-----------|-------------|
| **Readability** | Breaks a complex query into named, logical steps |
| **Reusability** | Define once, reference multiple times in the same query |
| **Maintainability** | Easier to update one CTE block than edit the same subquery in five places |
| **Debugging** | You can test each CTE individually by running its inner SELECT |
| **Recursion** | CTEs can reference themselves for hierarchical/tree data (e.g. org charts) |
| **No storage** | Unlike a view or temp table, a CTE uses no disk space |
| **Replaces complex nesting** | Avoids deeply nested subqueries that are hard to read |

---

## 5. CTE vs Subquery vs View

Understanding when to use each tool is important. Here is a side-by-side comparison:

| Feature | CTE (WITH Clause) | Inline Subquery | View |
|---|---|---|---|
| **Where defined** | Top of the query | Inside the query | Stored permanently in DB |
| **Scope / Lifetime** | Single query only | Single query only | Permanent until dropped |
| **Reusable across sessions** | No | No | Yes |
| **Can reference itself (recursion)** | Yes | No | No |
| **Readability** | High | Low (deeply nested) | High |
| **Requires DB permission to create** | No | No | Yes (CREATE VIEW) |
| **Best used for** | Complex, multi-step queries | Simple, one-time filters | Frequently reused logic |
| **Performance** | Similar to subquery | Similar to CTE | Can be optimized with indexes |

### When to pick which

- Use a **CTE** when your query is complex and you want it to be readable — especially if you need to reuse a result set within the same query.
- Use a **subquery** for simple, one-time filters that are short and easy to understand inline.
- Use a **view** when the same logic needs to be reused by many different queries, users, or applications.

---

## 6. Full Working Oracle SQL Example

This section walks through a complete, runnable Oracle SQL example from table creation to final output.

### Step 1 — Create the table

```sql
CREATE TABLE employees (
    emp_id     NUMBER PRIMARY KEY,
    emp_name   VARCHAR2(100),
    department VARCHAR2(50),
    salary     NUMBER(10, 2),
    hire_date  DATE
);
```

### Step 2 — Insert sample data

```sql
INSERT INTO employees VALUES (1,  'Alice Johnson',   'Engineering',  95000, DATE '2019-03-15');
INSERT INTO employees VALUES (2,  'Bob Smith',       'Marketing',    60000, DATE '2020-07-01');
INSERT INTO employees VALUES (3,  'Carol White',     'Engineering',  85000, DATE '2018-11-20');
INSERT INTO employees VALUES (4,  'David Brown',     'HR',           52000, DATE '2021-01-10');
INSERT INTO employees VALUES (5,  'Eva Martinez',    'Engineering', 110000, DATE '2017-06-05');
INSERT INTO employees VALUES (6,  'Frank Lee',       'Marketing',    72000, DATE '2019-09-22');
INSERT INTO employees VALUES (7,  'Grace Kim',       'HR',           48000, DATE '2022-04-30');
INSERT INTO employees VALUES (8,  'Henry Wilson',    'Finance',      91000, DATE '2016-12-01');
INSERT INTO employees VALUES (9,  'Isla Davis',      'Finance',      87000, DATE '2020-02-14');
INSERT INTO employees VALUES (10, 'James Taylor',    'Marketing',    65000, DATE '2021-08-19');

COMMIT;
```

### Step 3 — CTE query using `WITH`

**Goal:** Find all employees whose salary is above the average salary of their own department.

```sql
WITH dept_avg_salary AS (
    SELECT
        department,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
)
SELECT
    e.emp_id,
    e.emp_name,
    e.department,
    e.salary,
    ROUND(d.avg_salary, 2) AS dept_avg_salary
FROM employees e
JOIN dept_avg_salary d
    ON e.department = d.department
WHERE e.salary > d.avg_salary
ORDER BY e.department, e.salary DESC;
```

### Step 4 — Output table

| EMP_ID | EMP_NAME | DEPARTMENT | SALARY | DEPT_AVG_SALARY |
|--------|----------|------------|--------|-----------------|
| 1 | Alice Johnson | Engineering | 95000 | 96666.67 |
| 5 | Eva Martinez | Engineering | 110000 | 96666.67 |
| 8 | Henry Wilson | Finance | 91000 | 89000.00 |
| 6 | Frank Lee | Marketing | 72000 | 65666.67 |

> **Note:** Only employees earning above their department's average salary appear in the result.

### Step 5 — Step-by-step explanation

**Step A — The CTE block (`dept_avg_salary`)**

```sql
WITH dept_avg_salary AS (
    SELECT
        department,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
)
```

This block calculates the average salary for each department and temporarily names that result `dept_avg_salary`. Oracle runs this first and holds the result in memory.

**Step B — The main query**

```sql
SELECT
    e.emp_id,
    e.emp_name,
    e.department,
    e.salary,
    ROUND(d.avg_salary, 2) AS dept_avg_salary
FROM employees e
JOIN dept_avg_salary d
    ON e.department = d.department
WHERE e.salary > d.avg_salary
ORDER BY e.department, e.salary DESC;
```

This joins the `employees` table with the CTE result using the department name. The `WHERE` clause then filters to keep only employees who earn more than their department's average.

---

## 7. Practical Real-Life Examples

---

### Example 1 — Finding High-Salary Employees

**Task:** Find all employees earning more than 80,000.

```sql
WITH high_earners AS (
    SELECT emp_id, emp_name, department, salary
    FROM employees
    WHERE salary > 80000
)
SELECT *
FROM high_earners
ORDER BY salary DESC;
```

**Result:**

| EMP_ID | EMP_NAME | DEPARTMENT | SALARY |
|--------|----------|------------|--------|
| 5 | Eva Martinez | Engineering | 110000 |
| 1 | Alice Johnson | Engineering | 95000 |
| 8 | Henry Wilson | Finance | 91000 |
| 9 | Isla Davis | Finance | 87000 |
| 3 | Carol White | Engineering | 85000 |

---

### Example 2 — Department-wise Total Salary

**Task:** Calculate total salary spent per department and flag departments that exceed a budget of 200,000.

```sql
WITH dept_totals AS (
    SELECT
        department,
        COUNT(*)          AS total_employees,
        SUM(salary)       AS total_salary,
        ROUND(AVG(salary), 2) AS avg_salary
    FROM employees
    GROUP BY department
)
SELECT
    department,
    total_employees,
    total_salary,
    avg_salary,
    CASE
        WHEN total_salary > 200000 THEN 'Over Budget'
        ELSE 'Within Budget'
    END AS budget_status
FROM dept_totals
ORDER BY total_salary DESC;
```

**Result:**

| DEPARTMENT | TOTAL_EMPLOYEES | TOTAL_SALARY | AVG_SALARY | BUDGET_STATUS |
|------------|-----------------|--------------|------------|---------------|
| Engineering | 3 | 290000 | 96666.67 | Over Budget |
| Finance | 2 | 178000 | 89000.00 | Within Budget |
| Marketing | 3 | 197000 | 65666.67 | Within Budget |
| HR | 2 | 100000 | 50000.00 | Within Budget |

---

### Example 3 — Finding Top Customers by Purchase Amount

**Task:** From a sales table, find the top 3 customers by total purchase amount.

```sql
-- First, create the sales table
CREATE TABLE sales (
    sale_id     NUMBER PRIMARY KEY,
    customer_id NUMBER,
    customer_name VARCHAR2(100),
    amount      NUMBER(10, 2),
    sale_date   DATE
);

INSERT INTO sales VALUES (1, 101, 'Acme Corp',    15000, DATE '2024-01-10');
INSERT INTO sales VALUES (2, 102, 'Beta Ltd',      8500, DATE '2024-01-15');
INSERT INTO sales VALUES (3, 101, 'Acme Corp',    22000, DATE '2024-02-05');
INSERT INTO sales VALUES (4, 103, 'Gamma Inc',    31000, DATE '2024-02-20');
INSERT INTO sales VALUES (5, 102, 'Beta Ltd',     12000, DATE '2024-03-01');
INSERT INTO sales VALUES (6, 104, 'Delta LLC',     5500, DATE '2024-03-10');
INSERT INTO sales VALUES (7, 103, 'Gamma Inc',    18000, DATE '2024-03-15');
INSERT INTO sales VALUES (8, 105, 'Epsilon Co',   27000, DATE '2024-04-01');
COMMIT;

-- CTE query to find top 3 customers
WITH customer_totals AS (
    SELECT
        customer_id,
        customer_name,
        SUM(amount)   AS total_purchases,
        COUNT(*)      AS total_orders
    FROM sales
    GROUP BY customer_id, customer_name
),
ranked_customers AS (
    SELECT
        customer_id,
        customer_name,
        total_purchases,
        total_orders,
        RANK() OVER (ORDER BY total_purchases DESC) AS purchase_rank
    FROM customer_totals
)
SELECT *
FROM ranked_customers
WHERE purchase_rank <= 3
ORDER BY purchase_rank;
```

**Result:**

| CUSTOMER_ID | CUSTOMER_NAME | TOTAL_PURCHASES | TOTAL_ORDERS | PURCHASE_RANK |
|-------------|---------------|-----------------|--------------|---------------|
| 103 | Gamma Inc | 49000 | 2 | 1 |
| 105 | Epsilon Co | 27000 | 1 | 2 |
| 101 | Acme Corp | 37000 | 2 | 3 |

> **Note:** Two CTEs are chained here. `customer_totals` calculates totals, then `ranked_customers` adds a rank. This is a key strength of CTEs.

---

### Example 4 — Monthly Sales Report

**Task:** Generate a month-by-month sales summary for the year 2024.

```sql
WITH monthly_sales AS (
    SELECT
        TO_CHAR(sale_date, 'YYYY-MM')    AS sale_month,
        TO_CHAR(sale_date, 'Month YYYY') AS month_label,
        COUNT(*)                          AS total_orders,
        SUM(amount)                       AS total_revenue,
        ROUND(AVG(amount), 2)             AS avg_order_value
    FROM sales
    WHERE sale_date >= DATE '2024-01-01'
      AND sale_date <  DATE '2025-01-01'
    GROUP BY
        TO_CHAR(sale_date, 'YYYY-MM'),
        TO_CHAR(sale_date, 'Month YYYY')
)
SELECT
    month_label,
    total_orders,
    total_revenue,
    avg_order_value
FROM monthly_sales
ORDER BY sale_month;
```

**Result:**

| MONTH_LABEL | TOTAL_ORDERS | TOTAL_REVENUE | AVG_ORDER_VALUE |
|-------------|--------------|---------------|-----------------|
| January 2024 | 2 | 37000 | 18500.00 |
| February 2024 | 2 | 53000 | 26500.00 |
| March 2024 | 3 | 35500 | 11833.33 |
| April 2024 | 1 | 27000 | 27000.00 |

---

### Example 5 — Breaking a Complex Report into Readable Parts

**Task:** Generate a full employee performance summary that combines department stats, individual salary rank, and tenure (years at the company).

Without a CTE, this would be a single deeply nested query that is very hard to read. With CTEs, each step is named and clear:

```sql
WITH dept_stats AS (
    -- Step 1: Calculate statistics per department
    SELECT
        department,
        AVG(salary)  AS dept_avg,
        MAX(salary)  AS dept_max,
        MIN(salary)  AS dept_min
    FROM employees
    GROUP BY department
),
employee_rank AS (
    -- Step 2: Rank each employee within their department by salary
    SELECT
        emp_id,
        emp_name,
        department,
        salary,
        hire_date,
        RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS salary_rank
    FROM employees
),
tenure_calc AS (
    -- Step 3: Calculate how many years each employee has been with the company
    SELECT
        emp_id,
        FLOOR(MONTHS_BETWEEN(SYSDATE, hire_date) / 12) AS years_of_service
    FROM employees
)
-- Final query: Combine all three CTEs
SELECT
    er.emp_name,
    er.department,
    er.salary,
    er.salary_rank                            AS rank_in_dept,
    ROUND(ds.dept_avg, 2)                     AS dept_avg_salary,
    tc.years_of_service,
    CASE
        WHEN er.salary >= ds.dept_avg THEN 'Above Average'
        ELSE 'Below Average'
    END AS salary_status
FROM employee_rank  er
JOIN dept_stats     ds ON er.department = ds.department
JOIN tenure_calc    tc ON er.emp_id    = tc.emp_id
ORDER BY er.department, er.salary_rank;
```

This query is clean, readable, and self-documenting — each CTE block has a comment explaining exactly what it does.

---

## 8. How CTE Improves Query Readability

Consider finding employees above their department's average salary. Here is the same logic written two ways:

### Without CTE — deeply nested, hard to read

```sql
SELECT
    e.emp_name,
    e.department,
    e.salary
FROM employees e
WHERE e.salary > (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.department = e.department
)
ORDER BY e.department;
```

This works, but as complexity grows, nesting becomes very difficult to read and debug.

### With CTE — clear and structured

```sql
WITH dept_averages AS (
    SELECT department, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
)
SELECT
    e.emp_name,
    e.department,
    e.salary
FROM employees e
JOIN dept_averages d ON e.department = d.department
WHERE e.salary > d.avg_salary
ORDER BY e.department;
```

### Why the CTE version is better

| Concern | Without CTE | With CTE |
|---------|-------------|----------|
| **Readability** | Hard — logic buried inside the WHERE clause | Easy — logic separated and named |
| **Debugging** | Must extract the subquery manually to test it | Run the CTE block alone to test it |
| **Reusability** | Must copy-paste the subquery if used again | Reference the CTE name anywhere in the query |
| **Collaboration** | Teammates struggle to understand the intent | Self-documenting — CTE names explain the purpose |

---

## 9. When to Use CTE — and When Not To

### Use CTE when

- Your query has **multiple logical steps** that build on each other.
- You need to **reuse the same sub-result** more than once in a query.
- You want to make a **complex query easier to read and maintain**.
- You need **recursive queries** (e.g. org charts, folder trees, bill of materials).
- You are **ranking, partitioning, or windowing** data before filtering it.
- You are writing a **one-time report or analysis query**.

```sql
-- Good use case: multi-step analysis
WITH step1 AS (...),
     step2 AS (...),
     step3 AS (...)
SELECT * FROM step3;
```

### Avoid CTE when

- The logic is **simple enough for a one-line subquery** — a CTE would be overkill.
- The same logic needs to be **shared across many different queries or applications** — use a View instead.
- You are in a **performance-critical loop** where the CTE is materialised repeatedly — profile first.
- You need the result to **persist across sessions** — use a temporary table or a view.

```sql
-- Overkill: CTE for a trivially simple filter
-- Do this instead:
SELECT * FROM employees WHERE salary > 80000;

-- Not this:
WITH high_earners AS (SELECT * FROM employees WHERE salary > 80000)
SELECT * FROM high_earners;
```

---

## 10. Common Mistakes Beginners Make

### Mistake 1 — Putting a semicolon after the CTE definition

```sql
-- WRONG: Semicolon ends the statement prematurely
WITH dept_avg AS (
    SELECT department, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
); -- <-- ERROR: semicolon here breaks everything

SELECT * FROM dept_avg;
```

```sql
-- CORRECT: One semicolon at the very end
WITH dept_avg AS (
    SELECT department, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
)
SELECT * FROM dept_avg; -- <-- Only ONE semicolon here
```

---

### Mistake 2 — Trying to SELECT from CTE outside the query

```sql
-- WRONG: CTE only lives inside its own query
WITH temp_data AS (
    SELECT * FROM employees WHERE salary > 80000
)
SELECT * FROM temp_data;

-- This will ERROR — temp_data no longer exists:
SELECT * FROM temp_data;
```

> A CTE is temporary. It disappears as soon as the query finishes.

---

### Mistake 3 — Using ORDER BY inside a CTE (without ROWNUM or FETCH)

```sql
-- RISKY: Oracle may ignore ORDER BY inside a CTE
WITH sorted_employees AS (
    SELECT emp_name, salary
    FROM employees
    ORDER BY salary DESC -- This may be ignored inside a CTE
)
SELECT * FROM sorted_employees;

-- CORRECT: Apply ORDER BY in the final SELECT
WITH employee_data AS (
    SELECT emp_name, salary
    FROM employees
)
SELECT emp_name, salary
FROM employee_data
ORDER BY salary DESC; -- Order in the outer query
```

---

### Mistake 4 — Forgetting the comma between multiple CTEs

```sql
-- WRONG: Missing comma between CTEs
WITH cte_one AS (
    SELECT department FROM employees
)
-- Missing comma here!
cte_two AS (
    SELECT emp_name FROM employees
)
SELECT * FROM cte_one;
```

```sql
-- CORRECT: Comma after each CTE except the last
WITH cte_one AS (
    SELECT department FROM employees
),                         -- <-- comma here
cte_two AS (
    SELECT emp_name FROM employees
)                          -- <-- NO comma on the last one
SELECT * FROM cte_one;
```

---

### Mistake 5 — Referencing a CTE before it is defined

```sql
-- WRONG: second_cte references first_cte, but the order matters
WITH second_cte AS (
    SELECT * FROM first_cte -- ERROR: first_cte not defined yet
),
first_cte AS (
    SELECT * FROM employees
)
SELECT * FROM second_cte;
```

```sql
-- CORRECT: Define CTEs in the order they are needed
WITH first_cte AS (
    SELECT * FROM employees
),
second_cte AS (
    SELECT * FROM first_cte -- Now this works
)
SELECT * FROM second_cte;
```

---

### Mistake 6 — Using a CTE column name that doesn't exist

```sql
-- WRONG: The CTE doesn't produce a column called 'total'
WITH dept_data AS (
    SELECT department, SUM(salary) AS total_salary
    FROM employees
    GROUP BY department
)
SELECT department, total -- ERROR: column name is 'total_salary', not 'total'
FROM dept_data;
```

```sql
-- CORRECT: Use the exact alias defined in the CTE
SELECT department, total_salary
FROM dept_data;
```

---

## 11. Best Practices for Real Projects

### 1. Name your CTEs meaningfully

```sql
-- Bad: vague name
WITH temp1 AS (...)

-- Good: descriptive name
WITH dept_salary_summary AS (...)
```

### 2. Add comments to each CTE

```sql
WITH dept_salary_summary AS (
    -- Calculates total and average salary grouped by department
    SELECT
        department,
        SUM(salary)       AS total_salary,
        ROUND(AVG(salary), 2) AS avg_salary
    FROM employees
    GROUP BY department
)
```

### 3. One CTE = one responsibility

Each CTE should do exactly one logical job. Do not mix filtering, aggregation, and ranking all inside a single CTE.

```sql
-- Good: Each CTE has one clear job
WITH filtered_employees AS (
    SELECT * FROM employees WHERE hire_date >= DATE '2020-01-01'
),
dept_totals AS (
    SELECT department, SUM(salary) AS total FROM filtered_employees GROUP BY department
)
SELECT * FROM dept_totals;
```

### 4. Test each CTE individually during development

You can temporarily run just the CTE's inner SELECT to verify it returns what you expect before building the full query.

```sql
-- While developing, test this part first:
SELECT department, SUM(salary) AS total_salary
FROM employees
GROUP BY department;

-- Then wrap it in a CTE once you are confident it works.
```

### 5. Avoid unnecessary CTEs for simple queries

Not every query needs a CTE. Keep simple queries simple.

### 6. Use CTEs with window functions for ranking and analytics

CTEs pair very well with Oracle window functions like `RANK()`, `ROW_NUMBER()`, `LAG()`, and `LEAD()`.

```sql
WITH ranked_sales AS (
    SELECT
        customer_name,
        amount,
        ROW_NUMBER() OVER (ORDER BY amount DESC) AS rn
    FROM sales
)
SELECT customer_name, amount
FROM ranked_sales
WHERE rn <= 5; -- Top 5 sales
```

### 7. Profile performance when using large CTEs

In Oracle, CTEs are sometimes **inlined** (treated like a subquery) and sometimes **materialised** (computed once and cached). Use the `/*+ MATERIALIZE */` or `/*+ INLINE */` hints when you need to control this behaviour in performance-sensitive queries.

```sql
WITH dept_data AS (
    /*+ MATERIALIZE */
    SELECT department, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
)
SELECT * FROM dept_data;
```

---

## 12. Laravel + Oracle CTE Examples

Laravel developers working with Oracle databases (using the `yajra/laravel-oci8` package) often need to write raw CTEs. Here is how to do it correctly.

### Setup

Install the Oracle driver for Laravel:

```bash
composer require yajra/laravel-oci8
```

---

### Example 1 — Running a CTE with `DB::select()`

Use this approach when you need to **retrieve data** using a CTE.

```php
<?php

use Illuminate\Support\Facades\DB;

$results = DB::select("
    WITH dept_avg AS (
        SELECT department, ROUND(AVG(salary), 2) AS avg_salary
        FROM employees
        GROUP BY department
    )
    SELECT
        e.emp_name,
        e.department,
        e.salary,
        d.avg_salary
    FROM employees e
    JOIN dept_avg d ON e.department = d.department
    WHERE e.salary > d.avg_salary
    ORDER BY e.department
");

foreach ($results as $row) {
    echo $row->emp_name . ' - ' . $row->salary . PHP_EOL;
}
```

---

### Example 2 — CTE with bindings (safe parameter injection)

Always use **parameter binding** to prevent SQL injection — never concatenate user input directly into the query string.

```php
<?php

use Illuminate\Support\Facades\DB;

$minSalary = 80000;
$targetDept = 'Engineering';

$results = DB::select("
    WITH filtered_employees AS (
        SELECT emp_id, emp_name, department, salary
        FROM employees
        WHERE salary > :min_salary
          AND department = :department
    )
    SELECT *
    FROM filtered_employees
    ORDER BY salary DESC
", [
    'min_salary'  => $minSalary,
    'department'  => $targetDept,
]);

return response()->json($results);
```

---

### Example 3 — Monthly sales CTE in a Laravel Controller

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;

class SalesReportController extends Controller
{
    public function monthlyReport(Request $request)
    {
        $year = $request->input('year', date('Y'));

        $report = DB::select("
            WITH monthly_totals AS (
                SELECT
                    TO_CHAR(sale_date, 'MM')         AS month_num,
                    TO_CHAR(sale_date, 'Month YYYY') AS month_label,
                    COUNT(*)                          AS total_orders,
                    SUM(amount)                       AS total_revenue
                FROM sales
                WHERE TO_CHAR(sale_date, 'YYYY') = :year
                GROUP BY
                    TO_CHAR(sale_date, 'MM'),
                    TO_CHAR(sale_date, 'Month YYYY')
            )
            SELECT month_label, total_orders, total_revenue
            FROM monthly_totals
            ORDER BY month_num
        ", ['year' => $year]);

        return response()->json([
            'year'   => $year,
            'report' => $report,
        ]);
    }
}
```

---

### Example 4 — Top customers CTE as a reusable method

```php
<?php

namespace App\Repositories;

use Illuminate\Support\Facades\DB;

class CustomerRepository
{
    /**
     * Get the top N customers by total purchase amount.
     *
     * @param  int  $limit
     * @return array
     */
    public function getTopCustomers(int $limit = 10): array
    {
        return DB::select("
            WITH customer_totals AS (
                SELECT
                    customer_id,
                    customer_name,
                    SUM(amount)  AS total_purchases,
                    COUNT(*)     AS total_orders
                FROM sales
                GROUP BY customer_id, customer_name
            ),
            ranked_customers AS (
                SELECT
                    customer_id,
                    customer_name,
                    total_purchases,
                    total_orders,
                    RANK() OVER (ORDER BY total_purchases DESC) AS purchase_rank
                FROM customer_totals
            )
            SELECT *
            FROM ranked_customers
            WHERE purchase_rank <= :limit
            ORDER BY purchase_rank
        ", ['limit' => $limit]);
    }
}
```

**Usage in a controller:**

```php
$repo = new \App\Repositories\CustomerRepository();
$topCustomers = $repo->getTopCustomers(5);
return response()->json($topCustomers);
```

---

### Important notes for Laravel + Oracle developers

| Tip | Detail |
|-----|--------|
| **Use `DB::select()` for CTEs** | Laravel's query builder does not natively support CTEs, so raw SQL is the right approach |
| **Always use named bindings** | Oracle's PDO driver works best with named placeholders like `:param_name` |
| **Wrap in a repository or service** | Keep raw SQL out of controllers — use a repository or service class |
| **Return as collection** | Wrap results with `collect($results)` for Laravel collection methods |
| **Test with `DB::listen()`** | Use `DB::listen()` during development to log and inspect the exact SQL being sent to Oracle |

```php
// Debugging: log all queries during development
DB::listen(function ($query) {
    \Log::debug($query->sql, $query->bindings);
});
```

---

## Summary

| Topic | Key Takeaway |
|-------|-------------|
| **What is CTE** | A named, temporary result set defined with `WITH` at the start of a query |
| **Why use it** | Makes complex queries readable, reusable, and easier to debug |
| **Syntax** | `WITH name AS (SELECT ...) SELECT ... FROM name` |
| **vs Subquery** | CTE is named and reusable within the query; subquery is anonymous and inline |
| **vs View** | CTE is temporary (one query); View is permanent and stored in the DB |
| **Best for** | Multi-step logic, ranking, analytics, recursive data, complex reports |
| **Avoid when** | The query is simple or the logic needs to be shared across many queries |
| **In Laravel** | Use `DB::select()` with named bindings for CTE queries against Oracle |

---

> **Author note:** This document uses Oracle SQL syntax throughout. CTEs are part of the SQL standard (ISO/IEC 9075) and supported in Oracle, PostgreSQL, SQL Server, and MySQL 8.0+. Syntax is largely the same across databases, with minor differences.
