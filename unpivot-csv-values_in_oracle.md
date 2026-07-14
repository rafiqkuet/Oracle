# Unpivoting Comma-Separated Values in Oracle

**Technique:** XQuery sequence splitting via `XMLTABLE`
**Applies to:** Oracle Database 10gR2 and later
**Table:** `FACILITY_INFO`

---

## 1. Overview

This document describes a data extraction and normalization process for the `FACILITY_INFO` table. The Oracle SQL technique shown here parses a denormalized, comma-delimited string of reference values into individual, normalized rows — a "string unpivot."

Each extracted value retains its relationship to the parent record (`ID`, `FILE_ID`), allowing individual reference IDs to be filtered, joined, and aggregated independently.

The technique supports **both numeric** (`'339,456,1207'`) **and alphanumeric** (`'AA1,BB2,CC3,DD4'`) reference values. Section 3 provides variants for each case.

---

## 2. Source Schema Definition

The query operates on the `FACILITY_INFO` table:

| Column Name      | Data Type        | Description |
|------------------|------------------|-------------|
| `ID`             | `INTEGER`        | Primary identifier for the facility record. |
| `FILE_ID`        | `INTEGER`        | Identifier for the associated file. |
| `ECF_REFERENCES` | `VARCHAR2(281)`  | A denormalized, comma-separated list of reference IDs. Values may be numeric (`'339,456,1207'`) or alphanumeric (`'AA1,BB2,CC3,DD4'`). |

---

## 3. Query Implementation

### 3.1 General-Purpose Variant (Alphanumeric-Safe) — Recommended

Because `ECF_REFERENCES` may contain alphanumeric tokens such as `'AA1,BB2,CC3,DD4'`, the extracted value must be returned as a string, **not** cast with `TO_NUMBER` (which would raise `ORA-01722: invalid number` on non-numeric tokens).

```sql
SELECT
    f.ID,
    f.FILE_ID,
    TRIM(x.SEG_ID) AS SEG_ID
FROM
    FACILITY_INFO f,
    XMLTABLE(
        ('"' || REPLACE(f.ECF_REFERENCES, ',', '","') || '"')
        COLUMNS SEG_ID VARCHAR2(50) PATH '.'
    ) x
WHERE
    f.ID IN (645, 671);
```

Key points:

- The `COLUMNS ... PATH '.'` clause projects each sequence item directly as `VARCHAR2(50)`, avoiding an intermediate `XMLTYPE` value.
- `TRIM()` guards against incidental whitespace in the source string (e.g., `'AA1, BB2'`).
- This variant works identically for purely numeric data — the values are simply returned as strings.

### 3.2 Numeric-Only Variant

If a given dataset is guaranteed to contain only numeric tokens, the extracted value can be cast for numeric joins and comparisons:

```sql
SELECT
    ID,
    FILE_ID,
    TO_NUMBER(COLUMN_VALUE) AS SEG_ID
FROM
    FACILITY_INFO,
    XMLTABLE(('"' || REPLACE(ECF_REFERENCES, ',', '","') || '"'))
WHERE
    ID IN (645, 671);
```

> **Warning:** Do not use this variant against columns that may hold alphanumeric values. A single token like `'AA1'` will fail the entire query with `ORA-01722: invalid number`. If mixed data is possible, prefer §3.1 and apply a defensive cast downstream, e.g. `TO_NUMBER(SEG_ID DEFAULT NULL ON CONVERSION ERROR)` (Oracle 12.2+).

---

## 4. Mechanism of Action

The query relies on an XQuery sequence technique to split strings without heavy regular expressions (`REGEXP_SUBSTR`) or hierarchical queries (`CONNECT BY`).

1. **String Formatting (`REPLACE` + concatenation).** The raw string `'AA1,BB2,CC3,DD4'` is transformed into `'"AA1","BB2","CC3","DD4"'`. This is a valid XQuery expression: a comma-separated **sequence of string literals**. Alphanumeric content is fully supported because each token becomes a quoted literal.

2. **Row Generation (`XMLTABLE`).** `XMLTABLE` evaluates the dynamically generated XQuery sequence and emits **one row per sequence item**.

3. **Implicit Lateral Join.** Because `XMLTABLE` appears in the `FROM` clause alongside `FACILITY_INFO` and references its column, Oracle performs an implicit lateral join (equivalent to `CROSS APPLY`). Every generated row is correlated back to its parent record, preserving `ID` and `FILE_ID`.

4. **Value Projection.**
   - With a `COLUMNS ... PATH '.'` clause (§3.1), each item is projected directly to the declared SQL type (`VARCHAR2(50)`).
   - Without a `COLUMNS` clause (§3.2), the pseudocolumn `COLUMN_VALUE` is returned as `XMLTYPE` and must be explicitly converted (`TO_NUMBER`, `CAST`, or `.getStringVal()`).

---

## 5. Data Transformation Result

### Input State (Denormalized)

| ID  | FILE_ID | ECF_REFERENCES        |
|-----|---------|-----------------------|
| 645 | 784     | `339,456,1207`        |
| 671 | 814     | `1200,1214,1246,1236` |
| 702 | 890     | `AA1,BB2,CC3,DD4`     |

### Output State (Normalized)

| ID  | FILE_ID | SEG_ID |
|-----|---------|--------|
| 645 | 784     | 339    |
| 645 | 784     | 456    |
| 645 | 784     | 1207   |
| 671 | 814     | 1200   |
| 671 | 814     | 1214   |
| 671 | 814     | 1246   |
| 671 | 814     | 1236   |
| 702 | 890     | AA1    |
| 702 | 890     | BB2    |
| 702 | 890     | CC3    |
| 702 | 890     | DD4    |

---

## 6. Technical Notes

- **Performance.** `XMLTABLE`-based splitting is generally faster and less CPU-intensive than `REGEXP_SUBSTR` combined with `CONNECT BY LEVEL`, particularly on larger datasets, because it avoids repeated regex evaluation and row-multiplying hierarchical recursion per source row.

- **Alphanumeric support.** Quoted XQuery string literals accept letters, digits, and most punctuation. `TO_NUMBER` is the only numeric-specific step; omit it (or defer it) when values may be alphanumeric.

- **Limitations.**
  - **Embedded double quotes (`"`).** The generated XQuery sequence would be malformed if a token contains `"`. Pre-escape by doubling them: `REPLACE(ECF_REFERENCES, '"', '""')` applied *before* the comma replacement.
  - **XML-special characters.** Tokens containing `&` or `<` can break XQuery evaluation and would require escaping.
  - **NULL / empty source strings.** A `NULL` `ECF_REFERENCES` produces an empty sequence, so the parent row is **dropped** from the output. To preserve such rows, use an explicit `LEFT JOIN LATERAL` / `OUTER APPLY` (Oracle 12c+) or `COALESCE(ECF_REFERENCES, '')` handling.
  - **String length.** The concatenated XQuery expression must remain within `VARCHAR2` expression limits; this is not a concern for `VARCHAR2(281)` sources.

- **Alternatives (Oracle 12c+).** For JSON-friendly environments, `JSON_TABLE` offers a similar pattern with cleaner escaping semantics; in APEX-enabled databases, `APEX_STRING.SPLIT` provides a PL/SQL table function alternative. The `XMLTABLE` approach remains the most portable pure-SQL option back to 10gR2.

---

## 7. Reproducible Test Script

```sql
-- Setup
CREATE TABLE FACILITY_INFO
(
  ID              INTEGER,
  FILE_ID         INTEGER,
  ECF_REFERENCES  VARCHAR2(281 BYTE)
);

INSERT INTO FACILITY_INFO (ID, FILE_ID, ECF_REFERENCES)
VALUES (645, 784, '339,456,1207');

INSERT INTO FACILITY_INFO (ID, FILE_ID, ECF_REFERENCES)
VALUES (671, 814, '1200,1214,1246,1236');

INSERT INTO FACILITY_INFO (ID, FILE_ID, ECF_REFERENCES)
VALUES (702, 890, 'AA1,BB2,CC3,DD4');

COMMIT;

-- General-purpose (alphanumeric-safe) query
SELECT
    f.ID,
    f.FILE_ID,
    TRIM(x.SEG_ID) AS SEG_ID
FROM
    FACILITY_INFO f,
    XMLTABLE(
        ('"' || REPLACE(f.ECF_REFERENCES, ',', '","') || '"')
        COLUMNS SEG_ID VARCHAR2(50) PATH '.'
    ) x
WHERE
    f.ID IN (645, 671, 702);

/*
Expected result:

ID   FILE_ID  SEG_ID
---  -------  ------
645  784      339
645  784      456
645  784      1207
671  814      1200
671  814      1214
671  814      1246
671  814      1236
702  890      AA1
702  890      BB2
702  890      CC3
702  890      DD4
*/

-- Numeric-only variant (use only when all tokens are guaranteed numeric)
SELECT ID, FILE_ID, TO_NUMBER(COLUMN_VALUE) AS SEG_ID
FROM FACILITY_INFO,
     XMLTABLE(('"' || REPLACE(ECF_REFERENCES, ',', '","') || '"'))
WHERE ID IN (645, 671);

-- Cleanup
-- DROP TABLE FACILITY_INFO PURGE;
```
