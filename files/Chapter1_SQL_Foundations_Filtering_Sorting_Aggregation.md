# 📘 Chapter 1: SQL Foundations — SELECT, Filtering, Sorting & Aggregation
### *(SQLite Edition — converted from MSSQL)*

> 🧭 **Reference mapping:** Cheat Sheet Part 1 (Sections 1–9) + QnA Part 1 (Reference Chapter 3)
> 🗂️ **Setup used in notebook:**
> ```python
> %load_ext sql
> %sql sqlite:///mydata.db
> ```
> All queries below are written for **SQLite** (not MSSQL/T-SQL). Paste any query into a `%%sql` cell to run it against `mydata.db`.
>
> ⚠️ **Table note:** Your confirmed tables (from your notebooks) are `orders` and `returns`, with real lowercase column names like `order_id`, `customer_id`, `sales`, `profit`. Other tables used below (`employee`, `dept`, `customer`, `fd_customers`, `workers`, `FIREFIGHTERS`, `exams`) are generic SQL-practice tables carried over from the original cheat sheet for concept practice — they likely don't exist in your `mydata.db` yet. The `orders`/`returns` examples are written to match your real schema exactly.

---

## 1.1 SQL Logical Execution Order

SQL is **written** in one order but **executed** in a completely different order. This is the single most important mental model in SQL — almost every "why doesn't my query work" bug traces back to this.

```mermaid
flowchart LR
    A["1️⃣ FROM"] --> B["2️⃣ JOIN"] --> C["3️⃣ WHERE"] --> D["4️⃣ GROUP BY"] --> E["5️⃣ HAVING"] --> F["6️⃣ SELECT"] --> G["7️⃣ DISTINCT"] --> H["8️⃣ ORDER BY"] --> I["9️⃣ LIMIT"]
```

> ⚠️ **IMPORTANT TRAP — WHERE cannot use an aggregate function**
> ```sql
> -- ❌ WRONG — aggregate functions don't exist yet at WHERE stage
> SELECT category, SUM(sales) AS total_sales
> FROM orders
> WHERE SUM(sales) > 1000
> GROUP BY category;
> ```
> ```sql
> -- ✅ CORRECT — use HAVING to filter *after* aggregation
> SELECT category, SUM(sales) AS total_sales
> FROM orders
> GROUP BY category
> HAVING SUM(sales) > 1000;
> ```

> 💡 Note: MSSQL's execution-order diagrams usually end in `TOP`. In SQLite the equivalent final step is **`LIMIT`** (see §1.3) — same job, different keyword, different position in the query.

---

## 1.2 ⚠️ The Big Switch: MSSQL → SQLite (Chapter 1 relevant differences)

You previously wrote these cheat sheets against **MSSQL**. SQLite is a different engine with a smaller, stricter function set. Here are the differences that matter for *this* chapter — more engine-specific tables appear in later chapters.

| Concept | MSSQL (old) | SQLite (new) | Why |
|---|---|---|---|
| Top N rows | `SELECT TOP 5 *` (right after SELECT) | `SELECT * ... LIMIT 5` (at the **end**, after ORDER BY) | SQLite has no `TOP` keyword at all |
| Top N with paging | `OFFSET..FETCH NEXT` | `LIMIT 5 OFFSET 10` | Simpler single clause in SQLite |
| String concatenation | `'a' + 'b'` | `'a' \|\| 'b'` | `+` is only numeric in SQLite |
| Identifiers with spaces | `[Order Id]` | `"Order Id"` (brackets *also* still work, kept for MS-compatibility) | ANSI standard is double quotes |
| String literals | single **or** double quotes (depending on setting) | **single quotes ONLY** | In SQLite, double quotes always mean "identifier" — if no matching column/table exists it silently falls back to treating it as a string (a documented SQLite *misfeature*). Using `"text"` for a literal can cause silent bugs. Always use `'text'`. |
| `YEAR(date)` / `MONTH(date)` | built-in functions | ❌ Don't exist. Use `strftime('%Y', date)` / `strftime('%m', date)` | SQLite has no calendar-part extractor functions, only `strftime` |
| LIKE with `[abc]`, `[^abc]`, `[a-m]` | works inside `LIKE` | ❌ Does **not** work inside `LIKE`. Use `GLOB` instead (see §1.6) | SQLite's `LIKE` only understands `%` and `_` |

---

## 1.3 SELECT Basics

| Statement | Purpose |
|---|---|
| `SELECT *` | All columns |
| `SELECT DISTINCT city` | Unique values only |
| `SELECT col1, col2` | Specific columns |
| `SELECT COUNT(*)` | Count all rows |
| `SELECT COUNT(col)` | Count non-NULL values only |

**Top-N rows — the conversion you'll use constantly:**

```sql
-- MSSQL (old):  SELECT TOP 10 * FROM orders;
-- SQLite (new):
SELECT *
FROM orders
LIMIT 10;
```

```sql
-- Top N with ORDER BY — LIMIT always goes LAST, after ORDER BY
SELECT *
FROM employee
ORDER BY salary DESC
LIMIT 3;
```

> ✅ `LIMIT` also supports pagination: `LIMIT 10 OFFSET 20` → skip 20 rows, return the next 10.

---

## 1.4 WHERE Clause — Operators

Good news: every operator below is **standard SQL** and works identically in SQLite — no conversion needed.

| Operator | Meaning | Example |
|---|---|---|
| `=` | Equal | `age = 18` |
| `<>` / `!=` | Not equal | `age <> 18` |
| `>` | Greater than | `age > 18` |
| `<` | Less than | `age < 18` |
| `>=` | Greater or equal | `age >= 18` |
| `<=` | Less or equal | `age <= 18` |
| `BETWEEN` | Inclusive range | `age BETWEEN 18 AND 60` |
| `NOT BETWEEN` | Outside range | `age NOT BETWEEN 18 AND 60` |
| `IN` | Match in list | `city IN ('Pune','Delhi')` |
| `NOT IN` | Not in list | `city NOT IN ('Pune')` |
| `IS NULL` | Check NULL | `city IS NULL` |
| `IS NOT NULL` | Check NOT NULL | `city IS NOT NULL` |
| `EXISTS` | Subquery has rows | `EXISTS (SELECT 1 ...)` |

> ⚠️ **NULL trap (universal, not SQLite-specific):** if the list inside `NOT IN` contains even one `NULL`, the whole condition evaluates to `UNKNOWN` for every row, and **zero rows** are returned. Always filter NULLs out of the subquery feeding `NOT IN` (`WHERE col IS NOT NULL`). We'll see this again with `NOT IN` vs `NOT EXISTS` in Chapter 2.

---

## 1.5 AND / OR / NOT — Precedence

```
NOT  >  AND  >  OR
```

```
A OR B AND C   ⇔   A OR (B AND C)
```

> 💡 Use brackets for clarity: `(A OR B) AND C`

> ⚠️ **Real trap from the QnA bank** — "records in Technology **and** Furniture category for orders placed in 2020":
> ```sql
> -- Looks right, but isn't:
> SELECT * FROM orders
> WHERE category = 'Technology' OR category = 'Furniture'
>   AND YEAR(order_date) = 2020;          -- ❌ MSSQL function, also wrong logic
> ```
> Because `AND` binds tighter than `OR`, SQL reads this as:
> `category = 'Technology' OR (category = 'Furniture' AND YEAR(order_date) = 2020)`
> → **all** Technology rows (any year) leak in, only Furniture is restricted to 2020.
>
> ✅ Correct, with `IN` + brackets + the SQLite date fix (`YEAR()` → `strftime('%Y', ...)`):
> ```sql
> SELECT *
> FROM orders
> WHERE category IN ('Technology', 'Furniture')
>   AND strftime('%Y', order_date) = '2020';
> ```

---

## 1.6 ⚠️ LIKE & Wildcards — The Biggest Gotcha in This Chapter

This is the **#1 thing that breaks** when moving cheat-sheet SQL from MSSQL to SQLite.

| Wildcard | Works in SQLite `LIKE`? | SQLite equivalent |
|---|---|---|
| `%` — any number of chars | ✅ Yes, unchanged | `%` |
| `_` — exactly one char | ✅ Yes, unchanged | `_` |
| `[abc]` — char set | ❌ **No** — treated as literal characters, not a class | Use `GLOB` with `[abc]` |
| `[^abc]` — not in set | ❌ **No** | Use `GLOB` with `[^abc]` |
| `[a-m]` — range | ❌ **No** | Use `GLOB` with `[a-m]` |

SQLite has a **second** pattern-matching operator, `GLOB`, which *does* support character classes — but it uses Unix-glob symbols, not SQL ones:

| Need | SQLite `LIKE` (no brackets) | SQLite `GLOB` (brackets allowed) |
|---|---|---|
| Starts with S | `LIKE 'S%'` | `GLOB 'S*'` |
| Ends with a | `LIKE '%a'` | `GLOB '*a'` |
| Contains 'an' | `LIKE '%an%'` | `GLOB '*an*'` |
| Exactly 5 chars | `LIKE '_____'` | `GLOB '?????'` |
| Starts with S **or** M | not possible with LIKE | `GLOB '[SM]*'` |
| Starts with a digit | not possible with LIKE | `GLOB '[0-9]*'` |
| NOT starting with S/M | not possible with LIKE | `GLOB '[^SM]*'` |
| Letters A to M | not possible with LIKE | `GLOB '[A-M]*'` |

> ⚠️ **Case sensitivity flips!** SQLite's `LIKE` is case-**insensitive** for ASCII letters by default. `GLOB` is **always case-sensitive**. So `GLOB '[a-m]*'` and `GLOB '[A-M]*'` are *not* the same set — if you need both cases, write `[a-mA-M]`, or wrap the column in `LOWER()`/`UPPER()` first.

### Worked conversions from the QnA bank

**Q1 — customer name, 2nd char = 'a', 4th char = 'd'** *(no brackets → no change needed)*
```sql
SELECT *
FROM orders
WHERE customer_name LIKE '_a_d%';
```

**Q4 — name doesn't start with 'A' AND doesn't end with 'n'** *(no brackets → no change needed)*
```sql
SELECT *
FROM orders
WHERE customer_name NOT LIKE 'A%'
  AND customer_name NOT LIKE '%n';
```
> ✅ `NOT LIKE 'A%n'` would be **wrong** — that excludes only names that both start with A *and* end with n in one string; you need two separate `NOT LIKE` conditions joined with `AND`.

**Q11 — 2nd letter is a vowel, 3rd letter is NOT a vowel** *(has brackets → must switch to GLOB)*
```sql
-- MSSQL (old):  customer_name LIKE '_[aeiou][^aeiou]%'
-- SQLite (new):
SELECT customer_name
FROM orders
WHERE customer_name GLOB '?[aeiou][^aeiou]*';
```
> 💡 If names might start with an uppercase vowel too, either lowercase first — `LOWER(customer_name) GLOB '?[aeiou][^aeiou]*'` — or include both cases in the brackets: `[aeiouAEIOU]`.

**Q12 — name contains exactly two letter 'a's** *(uses LEN/REPLACE, not brackets — only a rename needed)*
```sql
-- MSSQL: LEN(customer_name) - LEN(REPLACE(customer_name,'a','')) = 2
SELECT customer_name
FROM orders
WHERE LENGTH(customer_name) - LENGTH(REPLACE(customer_name, 'a', '')) = 2;
-- ✅ LIKE '%a%a%' is wrong here — it means "at least two a's", not "exactly two".
```
*(We'll reuse this exact length-difference trick to count character occurrences in Chapter 4.)*

---

## 1.7 ORDER BY

```sql
ORDER BY sales DESC;
ORDER BY city ASC, sales DESC;   -- default is ASC if omitted
```
Fully standard — no SQLite differences.

---

## 1.8 Aggregate Functions

| Function | Purpose | Notes |
|---|---|---|
| `COUNT(*)` | Count rows | Counts **all** rows including NULLs |
| `COUNT(col)` | Count non-NULL | Ignores NULL |
| `SUM(col)` | Total | Ignores NULL |
| `AVG(col)` | Average | Ignores NULL |
| `MIN(col)` | Minimum | — |
| `MAX(col)` | Maximum | — |

> 💡 **Alternate way to compute AVG:**
> ```sql
> SELECT SUM(sales) / COUNT(sales) AS avg_sales   -- use COUNT(sales), not COUNT(*), so NULLs are excluded the same way AVG() excludes them
> FROM orders;
> ```

---

## 1.9 GROUP BY & HAVING

```sql
SELECT category, SUM(sales) AS total_sales
FROM orders
GROUP BY category
HAVING SUM(sales) > 10000;
```

| Feature | WHERE | HAVING |
|---|---|---|
| Filters | Rows | Groups |
| Applied | Before grouping | After grouping |
| Aggregate functions allowed? | ❌ No | ✅ Yes |

---

## 1.10 DISTINCT vs GROUP BY

| DISTINCT | GROUP BY |
|---|---|
| Removes duplicate rows | Creates groups |
| No aggregation | Usually paired with aggregation |
| Faster for simple uniqueness | More powerful — can aggregate per group |

---

## 1.11 Practice Bank — Chapter 3 QnA (SQLite-adapted)

> 🔄 = syntax had to change for SQLite · ⚠️ = trap/gotcha · ✅ = correct/recommended approach

**Q2 — Orders placed in Dec 2020** 🔄 (`MONTH()`/`YEAR()` don't exist in SQLite)
```sql
SELECT *
FROM orders
WHERE strftime('%m', order_date) = '12'
  AND strftime('%Y', order_date) = '2020';
```

**Q3 — ship_mode not in ('Standard Class','First Class') AND ship_date after Nov 2020**
```sql
SELECT *
FROM orders
WHERE ship_mode NOT IN ('Standard Class', 'First Class')
  AND ship_date >= '2020-12-01';
```

**Q5 — Negative profit orders**
```sql
SELECT * FROM orders WHERE profit < 0;
```

**Q6 — quantity < 3 OR profit = 0**
```sql
SELECT * FROM orders WHERE quantity < 3 OR profit = 0;
```

**Q7 — South region orders with a discount applied**
```sql
SELECT * FROM orders WHERE region = 'South' AND discount > 0;
-- ✅ Use the exact case 'South' as stored — SQLite string comparison is case-sensitive for '=' (unlike LIKE).
```

**Q8 — Top 5 order_ids by total sales, Furniture category** 🔄 (`TOP` → `LIMIT`)
```sql
SELECT order_id, SUM(sales) AS total_sales_per_order_id
FROM orders
WHERE category = 'Furniture'
GROUP BY order_id
ORDER BY total_sales_per_order_id DESC
LIMIT 5;
```

**Q9 — Technology AND Furniture records, year 2020 only** 🔄 ⚠️ (precedence trap + YEAR conversion)
```sql
SELECT *
FROM orders
WHERE category IN ('Technology', 'Furniture')
  AND strftime('%Y', order_date) = '2020';
```

**Q10 — order placed in 2020 but shipped in 2021** 🔄
```sql
SELECT *
FROM orders
WHERE strftime('%Y', order_date) = '2020'
  AND strftime('%Y', ship_date) = '2021';
```

**Q13 — Employees earning more than their own department's average salary** *(correlated subquery — fully portable)*
```sql
SELECT *
FROM employee t1
WHERE t1.salary > (
    SELECT AVG(salary)
    FROM employee t2
    WHERE t1.dept_id = t2.dept_id
);
```

**Q14 — Duplicate employee names**
```sql
SELECT emp_name, COUNT(*) AS name_count
FROM employee
GROUP BY emp_name
HAVING COUNT(*) > 1;
```

**Q15 — Los Angeles orders with sales above the city's average order amount**
```sql
SELECT *
FROM orders
WHERE city = 'Los Angeles'
  AND sales > (SELECT AVG(sales) FROM orders WHERE city = 'Los Angeles');
```

**Q16 — Top 3 highest-paid employees** 🔄
```sql
SELECT *
FROM employee
ORDER BY salary DESC
LIMIT 3;
```
> ✅ If you need top 3 **distinct** salaries (handling duplicates):
> ```sql
> SELECT DISTINCT salary
> FROM employee
> ORDER BY salary DESC
> LIMIT 3;
> ```

**Q17 — Cities where total FD amount exceeds 10 lakhs** *(HAVING required — WHERE can't filter an aggregate)*
```sql
SELECT city, SUM(fd_amount) AS total_fd
FROM fd_customers
GROUP BY city
HAVING SUM(fd_amount) > 1000000;
```

**Q18 — Subquery in the SELECT list (scalar per-row total)**
```sql
SELECT
    t1.customer_name,
    (SELECT SUM(sales) FROM orders t2 WHERE t1.customer_id = t2.customer_id) AS total_sales
FROM customers t1;
```

**Q19 — IN with a subquery**
```sql
SELECT *
FROM orders
WHERE customer_id IN (SELECT customer_id FROM customers WHERE city = 'Mumbai');
```

**Q20 — EXISTS with a correlated subquery**
```sql
SELECT *
FROM orders o
WHERE EXISTS (
    SELECT 1 FROM returns r WHERE r.order_id = o.order_id
);
```

---

## 1.12 Key Interview Tips ⭐

- ✅ `COUNT(column)` ignores NULL; `COUNT(*)` does not.
- ✅ Aggregate functions ignore NULL (except `COUNT(*)`).
- ✅ `AND` has higher precedence than `OR` — always bracket mixed conditions.
- ✅ `WHERE` cannot use aggregates — `HAVING` runs after `GROUP BY`.
- ✅ In SQLite, `LIKE` only understands `%` and `_`. The moment you need `[abc]`, `[^abc]`, or `[a-m]`, switch the **whole pattern** to `GLOB` (and remember it's case-sensitive).
- ✅ `TOP N` does not exist in SQLite — `LIMIT N` always goes at the very end of the query, after `ORDER BY`.
- ✅ `YEAR()` / `MONTH()` don't exist in SQLite — use `strftime('%Y', date)` / `strftime('%m', date)` (more on this in Chapter 4).
- ✅ String literals: single quotes only, in SQLite double quotes are for identifiers.

---

## 1.13 Quick Symbols Reference

| Symbol | Meaning |
|---|---|
| `=` | Equal |
| `<>` | Not equal |
| `>` / `<` | Greater than / Less than |
| `>=` / `<=` | Greater or equal / Less or equal |
| `!=` | Not equal |
| `%` | Any number of characters (LIKE) |
| `_` | Single character (LIKE) |
| `*` | Any number of characters (GLOB) |
| `?` | Single character (GLOB) |

---

➡️ **Next: Chapter 2 — NULL Handling, Subqueries & Comparison Operators (EXISTS / IN / ANY / ALL)**

⭐ *Practice regularly. Understand the logic, not just the syntax. Write efficient queries.* ⭐
