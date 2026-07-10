---
name: duckdb-sql
description: Use when writing SQL for DuckDB. Covers DuckDB-friendly SQL conventions, query formatting style, and preferred patterns.
---

# DuckDB SQL Conventions

## Formatting Style

Indent below each SQL clause keyword. Each clause keyword sits at the left margin; its contents are indented.

```sql
SELECT
    col_one: t.column_one,
    col_two: t.column_two,
    row_count: count(*)
FROM
    schema_name.table_name t
WHERE
    t.column_one = 'value'
    AND
    t.column_two > 100
    AND
    (t.status in ('active', 'pending') OR t.flag = true)
GROUP BY ALL
ORDER BY ALL;
```

- One column/expression per row in `SELECT`
- One filter per row in `WHERE`, `HAVING`, `ON`
- `AND` / `OR` on their own line between filters (not trailing on the previous line)
- Parenthesized sub-groups indented one level deeper
- Semicolon at the end of the last statement line
- 4-space indents, no tabs

## Naming & Aliases

### Prefix-style column aliases

Use `alias: expression` syntax (alias prefix before the expression). Always qualify column references with table aliases.

```sql
SELECT
    order_id: o.order_id,
    order_date: o.order_date,
    customer_name: c.customer_name,
    order_count: count(*)
FROM
    orders o
    INNER JOIN customers c ON o.customer_id = c.customer_id;
```

- Table aliases: short, meaningful (`o` for orders, `li` for line_items, `c` for customers)
- Column aliases: `snake_case`, match the source column name unless renaming for clarity
- Use `alias: expression` syntax for column aliases — do not use `as` or `=`

### Table aliases

Omit `as` between table name and alias.

```sql
FROM
    orders o
    INNER JOIN customers c ON o.customer_id = c.customer_id
```

### Identifier quoting

- Use double quotes `"` only when identifiers contain special characters or are case-sensitive
- Prefer unquoted `snake_case` identifiers everywhere
- Avoid quoted identifiers unless forced by the source schema
- CTEs follow the same alias rules: `WITH order_totals as (...)` — CTE definitions still use `as`

## DuckDB-Specific Patterns

### Prefer DuckDB syntax where available

| Feature | Do this | Don't do this |
|---|---|---|
| Grouping | `GROUP BY ALL` | `GROUP BY col1, col2, col3` |
| Ordering | `ORDER BY ALL` | `ORDER BY col1, col2` |
| Exclude columns | `SELECT * EXCLUDE (col_a, col_b)` | List all columns manually |
| Replace columns | `SELECT * REPLACE (coalesce(col, 0) as col)` | |
| Sample | `USING SAMPLE 10%` or `TABLESAMPLE 10%` | `LIMIT` for approximate results |
| Create table | `CREATE OR REPLACE TABLE` | `DROP TABLE IF EXISTS; CREATE TABLE` |
| Insert/append | `CREATE OR REPLACE TABLE t AS SELECT ...` or `INSERT INTO t SELECT ...` | |
| Exists | `CREATE TABLE t AS SELECT ... WHERE EXISTS (...)` | |
| List aggregation | `list(column)` or `array_agg(column)` | |

### Joins

```sql
SELECT
    order_id: o.order_id,
    line_total: li.line_total
FROM
    orders o
    INNER JOIN line_items li ON o.order_id = li.order_id
WHERE
    li.line_total > 100;
```

- Always qualify join columns with table aliases
- Use explicit `INNER JOIN`, `LEFT JOIN`, `CROSS JOIN` — never implicit commas
- One join per line
- `ON` clause on the same line as the join when brief; indent to a new line if complex

### Common Table Expressions (CTEs)

```sql
WITH
    order_totals as (
        SELECT
            order_id: o.order_id,
            total: sum(li.line_total)
        FROM
            orders o
            INNER JOIN line_items li ON o.order_id = li.order_id
        GROUP BY
            ALL
    ),
    high_value_orders as (
        SELECT
            order_id: ot.order_id,
            total: ot.total
        FROM
            order_totals ot
        WHERE
            ot.total > 1000
    )
SELECT
    order_id: hvo.order_id,
    total: hvo.total
FROM
    high_value_orders hvo
ORDER BY
    hvo.total desc;
```

- One CTE per logical step
- CTEs in dependency order (later CTEs reference earlier ones)
- `WITH` keyword on its own line
- CTE name and `as` on one line, the query body indented below
- CTE definitions use `as` (this is standard SQL syntax)

### Window Functions

```sql
SELECT
    order_id: t.order_id,
    order_date: t.order_date,
    amount: t.amount,
    rn: row_number() over (partition by t.customer_id order by t.order_date desc),
    customer_total: sum(t.amount) over (partition by t.customer_id)
FROM
    transactions t;
```

- Use `row_number()`, `rank()`, `dense_rank()`, `lag()`, `lead()`, `first_value()`, `last_value()`, `sum()`, `count()` as window functions
- One window function per line
- `over` keyword lower-case
- Break long `partition by` / `order by` clauses to multiple indented lines if needed

### Type casting

```sql
SELECT
    price_decimal: t.price::decimal(18, 2),
    created_date: t.created_at::date,
    count_big: t.count::bigint
FROM
    transactions t;
```

- Prefer `::` cast syntax over `CAST(x AS type)`
- Exception: use `CAST` for complex expressions where `::` precedence could be ambiguous

### Dates and Timestamps

```sql
SELECT
    month: date_trunc('month', t.created_at),
    days_since: date_diff('day', t.created_at, current_date),
    is_recent: t.created_at::date >= '2024-01-01'
FROM
    transactions t;
```

- Use `date_trunc`, `date_diff`, `date_add`, `extract` for date arithmetic
- Use `current_date`, `current_timestamp` over `now()` or `getdate()`
- ISO 8601 date literals: `'2024-01-15'`, `'2024-01-15 10:30:00'`

## Anti-Patterns (Avoid)

- `SELECT DISTINCT` as a band-aid for bad joins — find and fix the root cause
- `ORDER BY` without `LIMIT` in a CTE (use `ORDER BY` only in the final result or with `LIMIT`)
- `count(distinct x)` on high-cardinality columns — use `approx_count_distinct(x)` for large datasets
- Cartesian joins (missing `ON` clause)
- Implicit cross-database queries triggering data transfer
- Nested CTEs that could be flattened
- `NOT IN` with a subquery (use `NOT EXISTS` instead — `NOT IN` has unintuitive NULL behavior)
