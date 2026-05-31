# RANK() vs DENSE_RANK() in Oracle SQL

> A beginner-friendly guide to understanding window ranking functions in Oracle SQL — with real examples, output tables, and practical use cases.

---

## Table of Contents

- [Introduction](#introduction)
- [What is RANK()?](#what-is-rank)
- [What is DENSE\_RANK()?](#what-is-dense_rank)
- [Key Difference](#key-difference)
- [Syntax](#syntax)
- [Sample Table and Data](#sample-table-and-data)
- [SQL Queries Using RANK() and DENSE\_RANK()](#sql-queries-using-rank-and-dense_rank)
- [Output Tables](#output-tables)
- [What Happens With Ties?](#what-happens-with-ties)
- [Real-Life Use Cases](#real-life-use-cases)
- [When to Use RANK() vs DENSE\_RANK()](#when-to-use-rank-vs-dense_rank)
- [Comparison Table](#comparison-table)
- [Conclusion](#conclusion)

---

## Introduction

When working with data, we often need to **rank rows** based on a value — for example, ranking students by marks or employees by salary. Oracle SQL provides two powerful **window functions** for this purpose: `RANK()` and `DENSE_RANK()`.

Both functions assign a rank to each row, but they **handle ties differently**. Understanding this difference will help you choose the right function for the right situation.

---

## What is RANK()?

`RANK()` assigns a **sequential rank** to each row within a result set, ordered by a specified column. When two or more rows have the **same value (a tie)**, they receive the **same rank**, and the **next rank is skipped**.

**Example:** If two students both score 88 and share rank 2, the next student gets rank **4** (rank 3 is skipped).

---

## What is DENSE_RANK()?

`DENSE_RANK()` also assigns a sequential rank to each row. When there is a tie, rows receive the **same rank**, but the **next rank is NOT skipped** — ranks remain consecutive (dense).

**Example:** If two students both score 88 and share rank 2, the next student gets rank **3** (no gap).

---

## Key Difference

| Situation | RANK() | DENSE_RANK() |
|---|---|---|
| Two rows tied at rank 2 | Next rank is **4** (gap) | Next rank is **3** (no gap) |
| Rank sequence | May have gaps | Always consecutive |

---

## Syntax

### RANK() Syntax

```sql
RANK() OVER (
    [PARTITION BY partition_column]
    ORDER BY sort_column [ASC | DESC]
)
```

### DENSE_RANK() Syntax

```sql
DENSE_RANK() OVER (
    [PARTITION BY partition_column]
    ORDER BY sort_column [ASC | DESC]
)
```

> **Note:**
> - `PARTITION BY` is **optional**. Use it to reset the rank for each group (e.g., per department).
> - `ORDER BY` is **required**. It defines the column used for ranking.

---

## Sample Table and Data

We will use a **SCORES** table with student exam results.

### Create Table

```sql
CREATE TABLE scores (
    student_id   NUMBER,
    student_name VARCHAR2(50),
    score        NUMBER
);
```

### Insert Sample Data

```sql
INSERT INTO scores VALUES (1, 'Oliver',    95);
INSERT INTO scores VALUES (2, 'Amelia',    88);
INSERT INTO scores VALUES (3, 'George',    88);
INSERT INTO scores VALUES (4, 'Charlotte', 80);
INSERT INTO scores VALUES (5, 'Harry',     75);
COMMIT;
```

### Sample Data Preview

| student_id | student_name | score |
|---|---|---|
| 1 | Oliver | 95 |
| 2 | Amelia | 88 |
| 3 | George | 88 |
| 4 | Charlotte | 80 |
| 5 | Harry | 75 |

---

## SQL Queries Using RANK() and DENSE_RANK()

### Query: Using Both Functions Together

```sql
SELECT
    student_id,
    student_name,
    score,
    RANK() OVER (
        ORDER BY score DESC
    ) AS rank_position,
    DENSE_RANK() OVER (
        ORDER BY score DESC
    ) AS dense_rank_position
FROM scores
ORDER BY score DESC, student_id;
```

---

## Output Tables

### Result

| student_id | student_name | score | rank_position | dense_rank_position |
|---|---|---|---|---|
| 1 | Oliver | 95 | 1 | 1 |
| 2 | Amelia | 88 | 2 | 2 |
| 3 | George | 88 | 2 | 2 |
| 4 | Charlotte | 80 | **4** | **3** |
| 5 | Harry | 75 | **5** | **4** |

### Observation

- Amelia and George both scored **88**, so both get rank **2** in both functions.
- For the next student (Charlotte, score 80):
  - `RANK()` gives her **4** — because ranks 2 and 2 occupied two positions, so rank 3 is skipped.
  - `DENSE_RANK()` gives her **3** — no gap, ranks are always consecutive.

---

## What Happens With Ties?

When two or more rows have the **same value** in the `ORDER BY` column:

### With RANK()

- All tied rows receive the **same rank**.
- The next rank **jumps** by the number of tied rows.
- This can create **gaps** in the ranking sequence.

```
Rank sequence example: 1, 2, 2, 4, 5
                                  ↑
                              Rank 3 is skipped
```

### With DENSE_RANK()

- All tied rows receive the **same rank**.
- The next rank is simply **the next integer** — no gaps.
- The ranking sequence is always **consecutive**.

```
Rank sequence example: 1, 2, 2, 3, 4
                                  ↑
                              No gap — rank 3 is used
```

---

## Real-Life Use Cases

### 1. Ranking Students by Marks

```sql
SELECT
    student_name,
    marks,
    RANK()       OVER (ORDER BY marks DESC) AS rank_pos,
    DENSE_RANK() OVER (ORDER BY marks DESC) AS dense_rank_pos
FROM students
ORDER BY marks DESC;
```

- **Use case:** School or university leaderboard where you want to show student standings.
- **Best choice:** `DENSE_RANK()` — so that no rank number is skipped, and every student gets a fair position.

---

### 2. Ranking Employees by Salary

```sql
SELECT
    employee_name,
    department,
    salary,
    RANK()       OVER (PARTITION BY department ORDER BY salary DESC) AS rank_pos,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rank_pos
FROM employees
ORDER BY department, salary DESC;
```

- **Use case:** HR reporting to find the top earners within each department.
- **Best choice:** `DENSE_RANK()` when you want consistent rank numbers within each group; `RANK()` when you want to highlight exact position gaps caused by ties.

---

### 3. Ranking Products by Sales

```sql
SELECT
    product_name,
    total_sales,
    RANK()       OVER (ORDER BY total_sales DESC) AS rank_pos,
    DENSE_RANK() OVER (ORDER BY total_sales DESC) AS dense_rank_pos
FROM products
ORDER BY total_sales DESC;
```

- **Use case:** E-commerce or retail dashboards showing best-selling products.
- **Best choice:** `RANK()` if you want to reflect the true competitive gap between products.

---

### 4. Ranking Customers by Purchase Amount

```sql
SELECT
    customer_name,
    total_purchases,
    RANK()       OVER (ORDER BY total_purchases DESC) AS rank_pos,
    DENSE_RANK() OVER (ORDER BY total_purchases DESC) AS dense_rank_pos
FROM customers
ORDER BY total_purchases DESC;
```

- **Use case:** CRM or loyalty programs to identify top customers for rewards.
- **Best choice:** `DENSE_RANK()` — so that reward tiers (Gold, Silver, Bronze) are filled without gaps.

---

### 5. Ranking Players in a Tournament

```sql
SELECT
    player_name,
    total_points,
    RANK()       OVER (ORDER BY total_points DESC) AS rank_pos,
    DENSE_RANK() OVER (ORDER BY total_points DESC) AS dense_rank_pos
FROM tournament_results
ORDER BY total_points DESC;
```

- **Use case:** Sports leaderboards, gaming tournaments, or competition standings.
- **Best choice:** Depends on the rules. Use `RANK()` if tied players should share a rank and the next position is skipped (official sports style). Use `DENSE_RANK()` for continuous leaderboard positions.

---

## When to Use RANK() vs DENSE_RANK()

### Use `RANK()` when:

- You want to reflect **competitive gaps** in rankings.
- The **number of participants matters** (e.g., in a race, if two athletes tie for 2nd, the next gets 4th).
- You follow **official competition rules** where tied positions cause rank skipping.
- You want to know **exactly how many rows scored higher** than a given row.

### Use `DENSE_RANK()` when:

- You want **consecutive, gapless ranks**.
- You are building **leaderboards or reward tiers** where every level must be filled.
- You need to find the **Nth highest value** using a subquery (e.g., find the 3rd highest salary).
- Gaps in ranking would be **confusing or misleading** to end users.

---

## Comparison Table

| Feature | RANK() | DENSE_RANK() |
|---|---|---|
| Handles ties | Yes — same rank for tied rows | Yes — same rank for tied rows |
| Skips ranks after a tie | **Yes** — creates gaps | **No** — always consecutive |
| Rank sequence with ties | 1, 2, 2, **4**, 5 | 1, 2, 2, **3**, 4 |
| Total distinct ranks | May be less than row count | Equal to number of distinct values |
| Good for finding Nth value | Less reliable (gaps) | More reliable (no gaps) |
| Official competition style | Yes | Less common |
| Leaderboard / tiered rewards | Less suitable | More suitable |
| Oracle SQL support | Yes | Yes |
| Requires `OVER (ORDER BY ...)` | Yes | Yes |
| Supports `PARTITION BY` | Yes | Yes |

---

## Conclusion

Both `RANK()` and `DENSE_RANK()` are **window functions** in Oracle SQL that rank rows based on a specified order. The key distinction is simple:

- **`RANK()`** leaves **gaps** after tied values — reflecting the true positional gap.
- **`DENSE_RANK()`** keeps ranks **consecutive** — no positions are skipped.

Choosing the right function depends on your **business requirement**:

- If you're building a **competition leaderboard** or need to show **true positional gaps**, use `RANK()`.
- If you're designing **reward tiers**, **paginated rankings**, or finding the **Nth highest value**, use `DENSE_RANK()`.

Mastering these two functions will make your SQL queries more accurate, readable, and business-ready.

---

> **Author's Note:** This document uses **Oracle SQL syntax** only. The `OVER()` clause with `ORDER BY` is mandatory for both functions. The `PARTITION BY` clause is optional and resets the rank for each group.
