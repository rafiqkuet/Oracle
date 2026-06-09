# `LAST_VALUE` Analytic Function

## Purpose

The `LAST_VALUE` analytic function returns the last value from an ordered window of rows. It is useful when you need to display the final value in a group beside every row in that group.

A common use case is showing the lowest salesperson, latest status, final balance, last transaction value, or ending value beside each related row.

---

## Key Point

In Oracle, `LAST_VALUE` depends heavily on the window frame.

If you do not define the full window frame, Oracle may return the current row value instead of the true last value of the group.

To return the actual last value for every row in a partition, use:

```sql
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
```

---

## Basic Syntax

```sql
LAST_VALUE(column_name) OVER (
    PARTITION BY group_column
    ORDER BY sort_column
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

### Explanation

| Clause | Meaning |
|---|---|
| `LAST_VALUE(column_name)` | Returns the last value from the selected column within the analytic window. |
| `OVER (...)` | Defines the analytic window. |
| `PARTITION BY group_column` | Splits the data into separate groups. The function restarts for each group. |
| `ORDER BY sort_column` | Defines the order of rows inside each partition. |
| `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` | Expands the frame from the first row to the last row in the partition. |

---

## Sample Table

Assume we have a table named `sales`.

| sale_id | salesperson_name | region | total_sales |
|---:|---|---|---:|
| 1 | Oliver | North | 9500 |
| 2 | Amelia | North | 8700 |
| 3 | George | South | 8200 |
| 4 | Charlotte | South | 7600 |
| 5 | Harry | North | 6900 |
| 6 | Isla | South | 6400 |

---

## Objective

Show the lowest salesperson in each region beside every row.

The rows should be grouped by `region` and ordered by `total_sales` from highest to lowest. Because the lowest sales amount appears last in this order, `LAST_VALUE` returns the salesperson with the lowest sales in each region.

---

## Query

```sql
SELECT
    sale_id,
    salesperson_name,
    region,
    total_sales,
    LAST_VALUE(salesperson_name) OVER (
        PARTITION BY region
        ORDER BY total_sales DESC, sale_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS lowest_salesperson
FROM sales
ORDER BY region, total_sales DESC, sale_id;
```

---

## Expected Result

| sale_id | salesperson_name | region | total_sales | lowest_salesperson |
|---:|---|---|---:|---|
| 1 | Oliver | North | 9500 | Harry |
| 2 | Amelia | North | 8700 | Harry |
| 5 | Harry | North | 6900 | Harry |
| 3 | George | South | 8200 | Isla |
| 4 | Charlotte | South | 7600 | Isla |
| 6 | Isla | South | 6400 | Isla |

---

## How It Works

### 1. `PARTITION BY region`

Oracle first separates the rows into independent groups:

- `North`
- `South`

The `LAST_VALUE` calculation is performed separately inside each region.

### 2. `ORDER BY total_sales DESC, sale_id`

Inside each region, rows are sorted by `total_sales` in descending order.

For the `North` region:

| salesperson_name | total_sales |
|---|---:|
| Oliver | 9500 |
| Amelia | 8700 |
| Harry | 6900 |

The last row is `Harry`, so Oracle returns `Harry` for every row in the `North` partition.

For the `South` region:

| salesperson_name | total_sales |
|---|---:|
| George | 8200 |
| Charlotte | 7600 |
| Isla | 6400 |

The last row is `Isla`, so Oracle returns `Isla` for every row in the `South` partition.

### 3. Full Window Frame

This part is essential:

```sql
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
```

It tells Oracle to evaluate the function across the entire partition, from the first row to the last row.

Without this frame, `LAST_VALUE` usually works only up to the current row, which often produces unexpected results.

---

## Common Mistake in Oracle

### Query Without Full Window Frame

```sql
SELECT
    sale_id,
    salesperson_name,
    region,
    total_sales,
    LAST_VALUE(salesperson_name) OVER (
        PARTITION BY region
        ORDER BY total_sales DESC, sale_id
    ) AS lowest_salesperson
FROM sales
ORDER BY region, total_sales DESC, sale_id;
```

### Problem

This query may return the current row's `salesperson_name`, not the actual last salesperson in the region.

That happens because Oracle's default analytic window frame normally ends at the current row when an `ORDER BY` clause is used.

So the function sees rows only from the start of the partition up to the current row, not the full partition.

---

## Correct Pattern for Oracle

Use this pattern when you want the same final value repeated across the full group:

```sql
LAST_VALUE(column_name) OVER (
    PARTITION BY group_column
    ORDER BY sort_column
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

---

## Getting the Highest Salesperson Instead

If the rows are ordered by `total_sales DESC`, the highest salesperson appears first, not last. To return the highest salesperson in each region, use `FIRST_VALUE`:

```sql
SELECT
    sale_id,
    salesperson_name,
    region,
    total_sales,
    FIRST_VALUE(salesperson_name) OVER (
        PARTITION BY region
        ORDER BY total_sales DESC, sale_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS highest_salesperson
FROM sales
ORDER BY region, total_sales DESC, sale_id;
```

---

## Alternative: Use `LAST_VALUE` for Highest Salesperson

You can also reverse the sort order and use `LAST_VALUE`:

```sql
SELECT
    sale_id,
    salesperson_name,
    region,
    total_sales,
    LAST_VALUE(salesperson_name) OVER (
        PARTITION BY region
        ORDER BY total_sales ASC, sale_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS highest_salesperson
FROM sales
ORDER BY region, total_sales DESC, sale_id;
```

---

## Handling Ties

If multiple rows have the same `total_sales`, always add a secondary sorting column to make the result deterministic.

Example:

```sql
ORDER BY total_sales DESC, sale_id
```

Here, `sale_id` is used as a tie-breaker.

Without a tie-breaker, Oracle may return different results depending on the execution plan or row order.

---

## NULL Handling in Oracle

Oracle supports `RESPECT NULLS` and `IGNORE NULLS` with `LAST_VALUE`.

### Default Behavior

```sql
LAST_VALUE(salesperson_name) RESPECT NULLS OVER (...)
```

`RESPECT NULLS` is the default behavior. If the last value is `NULL`, Oracle returns `NULL`.

### Ignore NULL Values

```sql
LAST_VALUE(salesperson_name) IGNORE NULLS OVER (
    PARTITION BY region
    ORDER BY total_sales DESC, sale_id
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
) AS last_non_null_salesperson
```

Use `IGNORE NULLS` when you want Oracle to return the last non-null value in the window.

---

## Complete Oracle Demo Script

```sql
-- Drop table if it already exists
BEGIN
    EXECUTE IMMEDIATE 'DROP TABLE sales';
EXCEPTION
    WHEN OTHERS THEN
        IF SQLCODE != -942 THEN
            RAISE;
        END IF;
END;
/

-- Create sample table
CREATE TABLE sales (
    sale_id           NUMBER PRIMARY KEY,
    salesperson_name  VARCHAR2(100),
    region            VARCHAR2(50),
    total_sales       NUMBER
);

-- Insert sample data
INSERT INTO sales (sale_id, salesperson_name, region, total_sales)
VALUES (1, 'Oliver', 'North', 9500);

INSERT INTO sales (sale_id, salesperson_name, region, total_sales)
VALUES (2, 'Amelia', 'North', 8700);

INSERT INTO sales (sale_id, salesperson_name, region, total_sales)
VALUES (3, 'George', 'South', 8200);

INSERT INTO sales (sale_id, salesperson_name, region, total_sales)
VALUES (4, 'Charlotte', 'South', 7600);

INSERT INTO sales (sale_id, salesperson_name, region, total_sales)
VALUES (5, 'Harry', 'North', 6900);

INSERT INTO sales (sale_id, salesperson_name, region, total_sales)
VALUES (6, 'Isla', 'South', 6400);

COMMIT;

-- Query using LAST_VALUE with a full window frame
SELECT
    sale_id,
    salesperson_name,
    region,
    total_sales,
    LAST_VALUE(salesperson_name) OVER (
        PARTITION BY region
        ORDER BY total_sales DESC, sale_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS lowest_salesperson
FROM sales
ORDER BY region, total_sales DESC, sale_id;
```

---

## Best Practices

1. Always define the window frame when using `LAST_VALUE` with `ORDER BY`.
2. Use `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` when you need the true last value of the full partition.
3. Use `PARTITION BY` when the result should restart for each group.
4. Add a tie-breaker column in `ORDER BY` to make the result deterministic.
5. Use `IGNORE NULLS` if the last row may contain a null value and you need the last available non-null value.
6. Use `FIRST_VALUE` when the required value is naturally the first row in your sort order.

---

## Summary

Oracle's `LAST_VALUE` function is powerful, but it can be misleading if the window frame is not defined correctly.

For most reporting cases where the last value should appear beside every row in a group, use a full window frame:

```sql
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
```

This ensures that Oracle checks the complete partition rather than stopping at the current row.
