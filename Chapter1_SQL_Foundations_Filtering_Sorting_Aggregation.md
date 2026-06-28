

## 1.4 AND / OR / NOT Precedence

`NOT` binds tighter than `AND`, which binds tighter than `OR`. So `A OR B AND C` is read as `A OR (B AND C)`, not `(A OR B) AND C`. Bracket mixed conditions explicitly rather than relying on memorized precedence — both for your own clarity later and for whoever reviews the code next.

```sql
-- Intent: orders in Technology or Furniture, but only for 2020
-- Looks plausible, is wrong:
SELECT *
FROM orders
WHERE category = 'Technology' OR category = 'Furniture'
  AND strftime('%Y', order_date) = '2020';
```

Because `AND` binds tighter, this is actually evaluated as `category = 'Technology' OR (category = 'Furniture' AND strftime('%Y', order_date) = '2020')`. Every Technology row leaks in regardless of year; only Furniture gets restricted to 2020.

```sql
-- Correct: brackets force the intended grouping, IN replaces the repeated OR
SELECT *
FROM orders
WHERE category IN ('Technology', 'Furniture')
  AND strftime('%Y', order_date) = '2020';
```

Note this corrected version still wraps `order_date` in `strftime()`, so per the sargability point in 1.3 it is not index-friendly as written — a fully optimized version would use a range filter instead. Date functions get full treatment in Chapter 4; the precedence lesson here stands on its own regardless.

Interview angle: precedence bugs show up most often in dynamically generated SQL — ORM query builders, string-concatenated filters built up conditionally in application code — where nobody is looking directly at the final SQL text as it's written. The defensible practice at Lead level is to bracket every mixed `AND`/`OR` condition unconditionally, even when you're confident about precedence, and to actually read the generated SQL during code review rather than trusting the builder's output.

---

## 1.5 LIKE and Wildcards, and Where GLOB Takes Over

SQLite's `LIKE` understands exactly two wildcards: `%` for any sequence of characters, and `_` for exactly one character. Matching a literal pattern — starts with `S`, contains `an`, exactly five characters — needs no conversion at all from the MSSQL cheat sheet.

```sql
SELECT * FROM orders WHERE customer_name LIKE 'S%';
SELECT * FROM orders WHERE customer_name LIKE '%an%';
SELECT * FROM orders WHERE customer_name LIKE '_____';
```

The moment a pattern needs a character set — `[SM]`, `[^SM]`, `[A-M]` — `LIKE` in SQLite cannot express it; those brackets are treated as literal characters, not a class. The whole pattern has to move to `GLOB`, SQLite's second pattern operator, which understands the same bracket syntax MSSQL's `LIKE` used to, but with different symbols for the basic wildcards: `*` instead of `%`, `?` instead of `_`.

```sql
SELECT * FROM orders WHERE customer_name GLOB '[SM]*';     -- starts with S or M
SELECT * FROM orders WHERE customer_name GLOB '[0-9]*';    -- starts with a digit
SELECT * FROM orders WHERE customer_name GLOB '[^SM]*';    -- does not start with S or M
SELECT * FROM orders WHERE customer_name GLOB '[A-M]*';    -- starts with a letter A through M
```

Case sensitivity flips when you make that switch. `LIKE` in SQLite is case-insensitive for ASCII letters by default; `GLOB` is always case-sensitive, with no setting to change that. A pattern like `GLOB '[a-m]*'` will not match a name starting with a capital letter. If both cases matter, either widen the bracket — `[a-mA-M]` — or normalize the column first with `LOWER()`.

```sql
SELECT * FROM orders WHERE LOWER(customer_name) GLOB '[a-m]*';
```

Worked conversions from the question bank:

Second letter a vowel, third letter not a vowel — has brackets, so the whole pattern moves to `GLOB`:

```sql
-- MSSQL: customer_name LIKE '_[aeiou][^aeiou]%'
SELECT customer_name
FROM orders
WHERE customer_name GLOB '?[aeiou][^aeiou]*';
```

If uppercase vowels should also match, lowercase the column first or widen the bracket to `[aeiouAEIOU]`.

Name doesn't start with `A` and doesn't end with `n` — no brackets, no conversion needed:

```sql
SELECT *
FROM orders
WHERE customer_name NOT LIKE 'A%'
  AND customer_name NOT LIKE '%n';
```

A single `NOT LIKE 'A%n'` would be wrong here — that excludes only names that both start with A and end with n in the same string, not names that do either.

Name contains exactly two letter `a`'s — uses `LEN`/`REPLACE`, not brackets, so only a rename is needed:

```sql
-- MSSQL: LEN(customer_name) - LEN(REPLACE(customer_name,'a','')) = 2
SELECT customer_name
FROM orders
WHERE LENGTH(customer_name) - LENGTH(REPLACE(customer_name, 'a', '')) = 2;
```

`LIKE '%a%a%'` would be wrong here too — it means "at least two a's," not "exactly two."

Interview angle: there's a second, purely performance-driven reason to ask about wildcard placement. A pattern with a leading wildcard — `LIKE '%an%'`, `GLOB '*an*'` — can never use a standard B-tree index, because the index is sorted by the column's value starting from its first character, and the engine has no way to know where in that order a string containing "an" somewhere in the middle might fall. A trailing-only wildcard — `LIKE 'San%'` — can use the index, because every matching value sits in a contiguous range starting with "San." This is a property of how B-tree indexes work, not of any one database engine, and is one of the most commonly asked "why is this query slow" questions at senior level.

---

## 1.6 ORDER BY

Default sort direction is ascending if neither `ASC` nor `DESC` is specified, and multiple sort keys are evaluated left to right.

```sql
SELECT * FROM orders ORDER BY sales DESC;
SELECT * FROM orders ORDER BY city ASC, sales DESC;
```

Interview angle: `ORDER BY` on a large result set forces a full sort unless the engine can satisfy it directly from an index already stored in that order — worth checking with `EXPLAIN QUERY PLAN` if a query is slower than its filtering alone would justify. Pairing `ORDER BY` with `LIMIT` doesn't remove the sort by itself, but it does let the engine stop early once enough rows are produced, which is part of why bounding a sorted result with `LIMIT` is good practice whenever the full ordering isn't actually needed downstream.

---

## 1.7 Aggregate Functions

`COUNT(*)` counts every row, NULLs included. `COUNT(col)` counts only rows where `col` is not NULL. `SUM`, `AVG`, `MIN`, and `MAX` all ignore NULL values in the column they're aggregating.

```sql
SELECT
    COUNT(*)        AS total_rows,
    COUNT(discount) AS rows_with_discount,
    SUM(sales)       AS total_sales,
    AVG(sales)       AS avg_sales,
    MIN(profit)      AS min_profit,
    MAX(profit)      AS max_profit
FROM orders;
```

`AVG` can also be computed manually, which is occasionally useful when the average needs to be weighted or recomputed alongside other logic:

```sql
SELECT SUM(sales) / COUNT(sales) AS avg_sales   -- COUNT(sales), not COUNT(*), to match AVG's NULL handling
FROM orders;
```

Interview angle: a frequently asked, frequently misanswered question is whether `COUNT(1)` is faster than `COUNT(*)`. In every modern optimizer, including SQLite's, the two compile to an identical plan — there is no column being evaluated in either case, just a row counter. The distinction that actually matters is between `COUNT(*)`/`COUNT(1)` and `COUNT(some_column)`: the latter genuinely is a different operation, since it has to check each row for NULL in that specific column before counting it. Being able to correct the `COUNT(1)` myth confidently, while still correctly explaining when `COUNT(column)` legitimately produces a different number, signals real engine-level understanding rather than repeated folklore.

---

## 1.8 GROUP BY and HAVING

`WHERE` filters individual rows before grouping happens; `HAVING` filters whole groups after aggregation has already run. `HAVING` is the only place an aggregate function can be used as a filter condition.

```sql
SELECT category, SUM(sales) AS total_sales
FROM orders
GROUP BY category
HAVING SUM(sales) > 10000;
```

Interview angle: beyond the syntax rule, there's a real performance reason to prefer `WHERE` over `HAVING` whenever a filter doesn't actually depend on an aggregate. `WHERE` discards non-matching rows before the engine does the comparatively expensive work of grouping and aggregating; `HAVING` only gets to filter after every group has already been fully computed. A condition like `category = 'Furniture'` belongs in `WHERE` even in a query that also has `HAVING SUM(sales) > 10000` — writing it as `HAVING category = 'Furniture'` would still be logically correct, but it forces the engine to aggregate every category in the table before throwing most of the groups away. Spotting this kind of misplaced filter in someone else's query is a realistic Lead-level code review scenario, and naming the reason (rows discarded early vs. late) is what makes the answer land as senior rather than rote.

---

## 1.9 DISTINCT vs GROUP BY

`DISTINCT` removes duplicate rows from the result with no aggregation involved. `GROUP BY` forms groups, almost always to feed an aggregate function, and is the more powerful of the two when you actually need per-group calculations rather than just uniqueness.

```sql
SELECT DISTINCT city FROM orders;

SELECT category, COUNT(*) AS order_count
FROM orders
GROUP BY category;
```

It's commonly believed that `DISTINCT` is slower than an equivalent `GROUP BY`, but in most modern optimizers — SQLite included — the two compile to the same underlying plan when they express the same intent, because removing duplicates and forming groups both require sorting or hashing on the same columns. Rather than asserting a performance difference from memory, the senior-level move is to check directly:

```sql
EXPLAIN QUERY PLAN
SELECT DISTINCT category FROM orders;

EXPLAIN QUERY PLAN
SELECT category FROM orders GROUP BY category;
```

Comparing the two plan outputs, rather than repeating folklore about which one is faster, is the kind of habit interviewers are specifically listening for when they ask "is DISTINCT slower than GROUP BY."

---

## 1.10 Practice Bank — Chapter 3 Question Bank, SQLite-Adapted

Orders placed in December 2020. Conversion needed: `MONTH()`/`YEAR()` don't exist in SQLite.

```sql
SELECT *
FROM orders
WHERE strftime('%m', order_date) = '12'
  AND strftime('%Y', order_date) = '2020';
```

Ship mode not Standard or First Class, shipped after November 2020. No conversion needed.

```sql
SELECT *
FROM orders
WHERE ship_mode NOT IN ('Standard Class', 'First Class')
  AND ship_date >= '2020-12-01';
```

Negative-profit orders.

```sql
SELECT * FROM orders WHERE profit < 0;
```

Quantity under 3, or profit exactly zero.

```sql
SELECT * FROM orders WHERE quantity < 3 OR profit = 0;
```

South region orders with a discount applied. Exact case matters since `=` comparison is case-sensitive.

```sql
SELECT * FROM orders WHERE region = 'South' AND discount > 0;
```

Top 5 order IDs by total sales within the Furniture category. Conversion needed: `TOP` becomes `LIMIT`.

```sql
SELECT order_id, SUM(sales) AS total_sales_per_order_id
FROM orders
WHERE category = 'Furniture'
GROUP BY order_id
ORDER BY total_sales_per_order_id DESC
LIMIT 5;
```

Technology and Furniture records, year 2020 only. Conversion needed (`YEAR()`), and this is the precedence trap worked through in section 1.4.

```sql
SELECT *
FROM orders
WHERE category IN ('Technology', 'Furniture')
  AND strftime('%Y', order_date) = '2020';
```

Ordered in 2020 but shipped in 2021.

```sql
SELECT *
FROM orders
WHERE strftime('%Y', order_date) = '2020'
  AND strftime('%Y', ship_date) = '2021';
```

Employees earning more than their own department's average — a correlated subquery, fully portable as written.

```sql
SELECT *
FROM employee t1
WHERE t1.salary > (
    SELECT AVG(salary)
    FROM employee t2
    WHERE t1.dept_id = t2.dept_id
);
```

Duplicate employee names.

```sql
SELECT emp_name, COUNT(*) AS name_count
FROM employee
GROUP BY emp_name
HAVING COUNT(*) > 1;
```

Los Angeles orders with sales above the city's own average order amount.

```sql
SELECT *
FROM orders
WHERE city = 'Los Angeles'
  AND sales > (SELECT AVG(sales) FROM orders WHERE city = 'Los Angeles');
```

Top 3 highest-paid employees. Conversion needed: `TOP` becomes `LIMIT`.

```sql
SELECT *
FROM employee
ORDER BY salary DESC
LIMIT 3;
```

If duplicate salaries need to be handled so that genuinely distinct values are returned:

```sql
SELECT DISTINCT salary
FROM employee
ORDER BY salary DESC
LIMIT 3;
```

Cities where total FD amount exceeds ten lakh — `HAVING` is required here since `WHERE` cannot filter on an aggregate.

```sql
SELECT city, SUM(fd_amount) AS total_fd
FROM fd_customers
GROUP BY city
HAVING SUM(fd_amount) > 1000000;
```

Subquery placed directly in the SELECT list, computing a per-row scalar.

```sql
SELECT
    t1.customer_name,
    (SELECT SUM(sales) FROM orders t2 WHERE t1.customer_id = t2.customer_id) AS total_sales
FROM customers t1;
```

`IN` with a subquery.

```sql
SELECT *
FROM orders
WHERE customer_id IN (SELECT customer_id FROM customers WHERE city = 'Mumbai');
```

`EXISTS` with a correlated subquery.

```sql
SELECT *
FROM orders o
WHERE EXISTS (
    SELECT 1 FROM returns r WHERE r.order_id = o.order_id
);
```

---

## 1.11 Interview Perspective — Lead and Senior-Level Talking Points

- Logical execution order versus physical execution order: the optimizer can reorder steps, including pushing filters before joins, as long as the result matches what the logical order guarantees. Be ready to read `EXPLAIN QUERY PLAN` and reason about it rather than just reciting the order.
- `SELECT *` is a review flag for two concrete reasons: it blocks covering-index usage, and it silently changes shape if the underlying table's columns change later.
- Sargability: wrapping an indexed column in a function — `strftime('%Y', order_date)`, MSSQL's `YEAR(order_date)`, a leading-wildcard `LIKE`/`GLOB` pattern — defeats index usage. The fix is almost always a range condition on the raw column instead.
- `COUNT(1)` is not faster than `COUNT(*)` in any modern optimizer; the real distinction is `COUNT(*)` versus `COUNT(column)`, which differ in NULL handling, not performance.
- Prefer `WHERE` over `HAVING` for any condition that doesn't depend on an aggregate — it discards rows before the expensive grouping step instead of after.
- `DISTINCT` and `GROUP BY` frequently produce identical execution plans for the same intent; settle "which is faster" with `EXPLAIN QUERY PLAN`, not assumption.
- `NOT IN` against a subquery that can return NULL silently returns zero rows; `NOT EXISTS` does not have this failure mode and is the safer default for anti-joins.
- Defensive parenthesization of mixed `AND`/`OR` conditions matters most in dynamically generated SQL, where nobody is reading the final query text directly — bracket explicitly regardless of memorized precedence rules.

---

Next: Chapter 2 — NULL Handling, Subqueries and Comparison Operators (EXISTS / IN / ANY / ALL).
