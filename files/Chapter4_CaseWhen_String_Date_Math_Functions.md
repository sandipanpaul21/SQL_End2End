# 📕 Chapter 4: CASE WHEN, String Functions, Date/Time Functions & Type Conversion
### *(The Biggest MSSQL → SQLite Conversion Zone)*

> 🧭 **Reference mapping:** Cheat Sheet Part 1 §10 + Cheat Sheet Part 2 §22–23 + QnA Part 4 (Reference Chapter 6)
> 📌 Continued from Chapter 3. This is the densest conversion chapter — MSSQL's date/string function library is large and SQLite's is deliberately minimal, so almost every function here needs a rewrite. Every conversion below was **tested against a real SQLite 3.45 engine** before being written here, not just recalled from memory.

---

## 4.1 CASE WHEN — Basic Syntax *(fully portable, no changes)*

```sql
CASE
    WHEN condition1 THEN result1
    WHEN condition2 THEN result2
    ELSE result_default
END AS column_name
```
> ⚠️ **Order matters.** SQL evaluates `WHEN` clauses top-to-bottom and stops at the **first** TRUE condition — it never "falls through" to check later, more specific conditions after an earlier broad one matches.

```sql
-- Profit grouping
SELECT order_id, profit, sales,
    CASE
        WHEN profit < 0   THEN 'loss'
        WHEN profit <= 50 THEN 'low profit'
        WHEN profit <= 100 THEN 'medium profit'
        WHEN profit <= 150 THEN 'high profit'
        ELSE 'very high profit'
    END AS profit_grouping
FROM orders;
```

```sql
-- Count loss-making orders
SELECT SUM(CASE WHEN profit < 0 THEN 1 ELSE 0 END) AS loss_orders
FROM orders;
```

```sql
-- Average profit of only PROFITABLE orders
-- AVG() ignores NULL automatically, so omitting ELSE works as a filter:
SELECT AVG(CASE WHEN profit > 0 THEN profit END) AS avg_positive_profit
FROM orders;
```

```sql
-- Flag creation: 1 if profitable, else 0
SELECT order_id, profit,
    CASE WHEN profit > 0 THEN 1 ELSE 0 END AS profit_flag
FROM orders;
```

```sql
-- NULL handling with CASE WHEN
SELECT
    CASE WHEN customer_name IS NULL THEN 'missing name' ELSE customer_name END AS customer_status
FROM orders;
```

```sql
-- FIREFIGHTERS-style single-value aggregate (generic practice table — see Chapter 1 table note)
SELECT AVG(PeopleSaved) AS "AVG(PeopleSaved)" FROM FIREFIGHTERS WHERE CountryCode = 'PM';
SELECT SUM(PeopleSaved) AS "SUM(PeopleSaved)" FROM FIREFIGHTERS WHERE CountryCode = 'PG';
```
> 💡 `GROUP BY` isn't needed here — we're filtering down to a single country code, so the whole filtered set *is* the group.
> ✅ **Always use single quotes for string literals** — `'PM'` not `"PM"`. In SQLite, double quotes are reserved for identifiers; `"PM"` would either error or (worse) silently behave like a column reference fallback.

---

## 4.2 ⚠️ Date & Time Functions — Read This Before Writing Any Date Logic

MSSQL has a rich, dedicated date-function library. **SQLite has almost none of it** — instead it has 5 general-purpose functions (`date()`, `time()`, `datetime()`, `julianday()`, `strftime()`) that do everything via modifiers and format codes. Every date column in `orders` (`order_date`, `ship_date`) is stored as ISO-8601 text (`'YYYY-MM-DD'`), which is exactly what these functions expect.

| Need | MSSQL (old) | SQLite (new) — ✅ verified |
|---|---|---|
| Current date+time | `GETDATE()` | `DATETIME('now')` *(UTC)* or `DATETIME('now','localtime')` |
| Current date only | `CAST(GETDATE() AS DATE)` | `DATE('now')` |
| Current timestamp | `CURRENT_TIMESTAMP` | `CURRENT_TIMESTAMP` (works, but returns UTC text) |
| Add N days/months/years | `DATEADD(DAY, n, date)` | `DATE(date, '+n day')`, `'+n month'`, `'+n year'` |
| Difference in **days** | `DATEDIFF(DAY, d1, d2)` | `CAST(julianday(d2) - julianday(d1) AS INTEGER)` |
| Difference in **years** | `DATEDIFF(YEAR, d1, d2)` | `CAST(strftime('%Y',d2) AS INT) - CAST(strftime('%Y',d1) AS INT)` |
| Extract year/month/day | `DATEPART(...)` / `EXTRACT(... FROM ...)` | `strftime('%Y', date)`, `strftime('%m', date)`, `strftime('%d', date)` *(returns TEXT — `CAST(... AS INTEGER)` if needed)* |
| End of month | `EOMONTH(date)` | `DATE(date, 'start of month', '+1 month', '-1 day')` |
| Day name (e.g. "Sunday") | `DATENAME(WEEKDAY, date)` | `strftime('%w', date)` → `'0'`–`'6'` (Sun=0); map to a name with `CASE` (no built-in name function) |
| Build a date from parts | `DATEFROMPARTS(y,m,d)` | `DATE(printf('%04d-%02d-%02d', y, m, d))` |
| String → date | `CONVERT(DATE, str, style)` | `DATE(str)` *(works directly if `str` is already `'YYYY-MM-DD'`)* |
| Custom formatting | `FORMAT(date, 'fmt')` | `strftime('fmt', date)` *(uses `%Y %m %d %H %M %S` codes, different syntax from MSSQL's `FORMAT`)* |

```sql
-- EXTRACT(YEAR FROM date) does NOT work in SQLite at all — confirmed syntax error.
-- Always use strftime() instead:
SELECT strftime('%Y', order_date) AS order_year FROM orders;
```

> ⚠️ **Gotcha confirmed by testing:** `date()` month/year arithmetic does **not clamp** to the last valid day — it overflows instead.
> ```sql
> SELECT date('2024-01-31', '+1 month');   -- returns '2024-03-02', NOT '2024-02-29'!
> ```
> Because February has no 31st, SQLite adds the 2 extra days into March. If you need calendar-safe "end of next month" logic, combine with the `'start of month'` modifier rather than adding months directly to a day-31 date.

```sql
-- Difference in days (DATEDIFF replacement) — verified: Jan 1 → Jan 10 = 9 (not 10)
SELECT CAST(julianday('2024-01-10') - julianday('2024-01-01') AS INTEGER);  -- 9
```

```sql
-- End of month — handles leap years correctly (verified: Feb 2024 → '2024-02-29')
SELECT DATE('2024-02-15', 'start of month', '+1 month', '-1 day');
```

---

## 4.3 Business Days Between Two Dates — Why the Old Formula Breaks, and the SQLite-Native Fix

MSSQL's classic formula (`DATEDIFF(DAY,...) - DATEDIFF(WEEK,...) * 2`) relies on `DATEDIFF(WEEK,...)`, which counts calendar week-boundaries crossed — a calculation SQLite has no direct equivalent for. Rather than hand-rolling fragile boundary math, SQLite gives you something better: a **recursive CTE** that walks day-by-day and simply counts weekdays. This is more transparent and easier to trust than a closed-form formula.

```sql
-- ✅ Verified working: counts weekdays (Mon–Fri) between order_date and ship_date, per row
WITH RECURSIVE date_walk AS (
    SELECT row_id, order_date AS d, ship_date
    FROM orders
    UNION ALL
    SELECT row_id, date(d, '+1 day'), ship_date
    FROM date_walk
    WHERE d < ship_date
)
SELECT row_id,
       COUNT(*) AS business_days        -- counts only Mon–Fri dates in the range, inclusive
FROM date_walk
WHERE CAST(strftime('%w', d) AS INTEGER) NOT IN (0, 6)   -- exclude Sunday(0) and Saturday(6)
GROUP BY row_id;
```
> 💡 Tested example: `2024-01-01` (Mon) → `2024-01-10` (Wed) = **8** business days; `2024-02-01` (Thu) → `2024-02-05` (Mon) = **3** business days. Both match the expected manual count.
> ⚠️ `WITH RECURSIVE` is a genuinely useful SQLite feature with no MSSQL cheat-sheet equivalent shown earlier — it's the cleanest way to do *any* "walk through a date range" problem.

---

## 4.4 ⚠️ String Functions — Conversion Table

| MSSQL (old) | SQLite (new) | Verified Behavior |
|---|---|---|
| `LEN(str)` | `LENGTH(str)` | `LEN()` doesn't exist — confirmed error |
| `CHAR_LENGTH(str)` | `LENGTH(str)` | SQLite has only one length function |
| `UPPER(str)` / `LOWER(str)` | same | ✅ no change |
| `LTRIM` / `RTRIM` / `TRIM` | same | ✅ no change |
| `SUBSTRING(str, start, len)` | `SUBSTR(str, start, len)` | rename only, same 1-indexed behavior |
| `LEFT(str, n)` | `SUBSTR(str, 1, n)` | `LEFT()` doesn't exist — confirmed error |
| `RIGHT(str, n)` | `SUBSTR(str, -n)` | `RIGHT()` doesn't exist, but `SUBSTR` with a **negative start** counts from the end — confirmed: `SUBSTR('Hello',-3)` → `'llo'` |
| `REPLACE(str, a, b)` | same | ✅ no change. ⚠️ Any `NULL` argument → whole result is `NULL` (confirmed) |
| `REVERSE(str)` | ❌ **no built-in function at all** | Confirmed missing. If truly needed, use a small recursive CTE (see box below) |
| `CONCAT(a, b, c)` | `a \|\| b \|\| c` | The `\|\|` operator always works. **Bonus:** modern SQLite (≥ 3.44) *also* has `CONCAT()`/`CONCAT_WS()` natively — confirmed working — but `\|\|` is the safe, version-proof default |
| `STRING_AGG(col, sep) WITHIN GROUP (ORDER BY x)` | `GROUP_CONCAT(col, sep)` | See §4.5 below — ordering needs care |
| `CHARINDEX(substr, str)` | `INSTR(str, substr)` | ⚠️ **Argument order is reversed!** MSSQL: needle-then-haystack. SQLite: haystack-then-needle. Confirmed via testing. |
| `SPLIT(...)` (DB-specific) | no built-in split-to-rows | Needs `json_each()` on a JSON array, a recursive CTE, or splitting in Python before/after the query |

> 💡 **No built-in REVERSE — workaround (rarely needed in practice):**
> ```sql
> WITH RECURSIVE rev(s, i, out) AS (
>     SELECT 'Hello', LENGTH('Hello'), ''
>     UNION ALL
>     SELECT s, i - 1, out || SUBSTR(s, i, 1) FROM rev WHERE i > 0
> )
> SELECT out FROM rev ORDER BY i LIMIT 1;   -- → 'olleH'
> ```

---

## 4.5 STRING_AGG → GROUP_CONCAT — Getting the Order Right

```sql
-- ✅ Universally compatible method: pre-sort in a subquery, THEN aggregate
SELECT manager_name, GROUP_CONCAT(emp_name, ',') AS emp_list
FROM (
    SELECT e2.emp_name AS manager_name, e1.emp_name AS emp_name, e1.salary
    FROM employee e1
    INNER JOIN employee e2 ON e1.manager_id = e2.emp_id
    ORDER BY manager_name, e1.salary
)
GROUP BY manager_name;
```
```sql
-- ✅ Also works on modern SQLite (≥3.44) — inline ORDER BY inside the aggregate, confirmed working:
SELECT e2.emp_name AS manager_name,
       GROUP_CONCAT(e1.emp_name, ',' ORDER BY e1.salary) AS emp_list
FROM employee e1
INNER JOIN employee e2 ON e1.manager_id = e2.emp_id
GROUP BY e2.emp_name;
```
> 💡 If you're unsure which SQLite version your environment ships, the subquery method is the safe default — it works on every version.

---

## 4.6 Math Functions — Mostly Good News, Two Real Traps

Most MSSQL math function names carry straight over **unchanged** in SQLite 3.35+ (which includes the math-functions extension by default in modern builds, including the one used in this notebook). All of these were individually tested and confirmed working:

| Function | Works as-is in SQLite? | Example |
|---|---|---|
| `ROUND(x, n)` | ✅ | `ROUND(12.345, 2)` → `12.35` |
| `CEILING(x)` / `CEIL(x)` | ✅ both | `CEILING(4.2)` → `5` |
| `FLOOR(x)` | ✅ | `FLOOR(4.9)` → `4` |
| `ABS(x)` | ✅ | `ABS(-10)` → `10` |
| `POWER(x,y)` / `POW(x,y)` | ✅ both | `POWER(2,3)` → `8` |
| `SQRT(x)` | ✅ | `SQRT(16)` → `4` |
| `MOD(x,y)` | ✅ (also `x % y` for integers) | `MOD(10,3)` → `1` |
| `SIGN(x)` | ✅ | `SIGN(-5)` → `-1`, `SIGN(0)` → `0` |
| `EXP(x)` | ✅ | `EXP(1)` → `2.718...` |
| `RADIANS(x)` / `DEGREES(x)` | ✅ both | `RADIANS(180)` → `3.14159` |

### ⚠️ Trap 1 — `LOG()` means something different in each engine

| | MSSQL `LOG(x)` | SQLite `LOG(x)` |
|---|---|---|
| Default base | **Natural log** (base *e*) | **Base 10** (same as `LOG10(x)`) — confirmed: `LOG(100)` → `2.0` |
| Natural log | `LOG(x)` | `LN(x)` — confirmed working |
| Two-argument form | `LOG(x, base)` — value first, base second | `LOG(base, x)` — **base first, value second** — confirmed: `LOG(2,8)` → `3.0` (log base 2 of 8) |

> ⚠️ This is a silent-bug risk: the function name is identical but the **default base and the two-argument order are both reversed**. Always double check which engine you're targeting.

### ⚠️ Trap 2 — `TRUNCATE(x, n)` has no 2-argument equivalent

SQLite's `TRUNC(x)` only takes **one** argument (truncate toward zero to the nearest integer) — confirmed: `TRUNC(12.3456, 2)` errors with "wrong number of arguments". For truncating to N decimal places, use this workaround (confirmed working):

```sql
-- TRUNCATE(12.3456, 2) equivalent → 12.34
SELECT CAST(12.3456 * POWER(10, 2) AS INTEGER) / POWER(10, 2);
```

---

## 4.7 Type Conversion

| MSSQL | SQLite | Notes |
|---|---|---|
| `CAST(expr AS datatype)` | `CAST(expr AS datatype)` | ✅ works the same — but SQLite is dynamically typed (type *affinity*, not strict types), so behavior on bad input differs (see below) |
| `CONVERT(datatype, expr, style)` | — | Doesn't exist; use `CAST` (no "style" formatting codes — build the string manually with `strftime`/`printf` if needed) |
| `TRY_CAST` / `TRY_CONVERT` | — | Don't exist — confirmed syntax error. **But you don't need them:** plain `CAST` in SQLite never throws on bad input |

```sql
SELECT CAST('abc' AS INTEGER);     -- → 0   (no error — SQLite extracts what it can)
SELECT CAST('12abc' AS INTEGER);   -- → 12  (takes the leading numeric portion)
```
> 💡 Since SQLite's `CAST` is already "safe" by default (never errors, just does its best), `TRY_CAST`'s entire purpose in MSSQL — avoiding a thrown error — is unnecessary here. Just use `CAST` directly.

---

## 4.8 Practice Bank — Chapter 6 QnA (SQLite-adapted)

**Employee / manager DOB difference (in days)** 🔄 `DATEDIFF` → `julianday`
```sql
SELECT t1.emp_name AS employee_name,
       t2.emp_name AS manager_name,
       CAST(julianday(t2.dob) - julianday(t1.dob) AS INTEGER) AS days_difference
FROM employee t1
INNER JOIN employee t2 ON t1.manager_id = t2.emp_id
WHERE t1.dob < t2.dob;
```
> 💡 Positive because the employee's DOB (earlier date) is subtracted from the manager's DOB.

**Sub-categories with zero return orders in November (any year)** 🔄 `MONTH()` → `strftime`
```sql
SELECT DISTINCT sub_category
FROM orders
WHERE sub_category NOT IN (
    SELECT DISTINCT t1.sub_category
    FROM orders t1
    INNER JOIN returns t2 ON t1.order_id = t2.order_id
    WHERE strftime('%m', t1.order_date) = '11'
);
```

**Order IDs where exactly one product was bought** *(portable)*
```sql
SELECT order_id, COUNT(*) AS product_count
FROM orders
GROUP BY order_id
HAVING COUNT(*) = 1;
```

**Category: total sales vs. sales of returned orders** *(portable — LEFT JOIN + CASE)*
```sql
SELECT t1.category,
       SUM(t1.sales) AS total_sales,
       SUM(CASE WHEN t2.order_id IS NOT NULL THEN t1.sales ELSE 0 END) AS returned_sales
FROM orders t1
LEFT JOIN returns t2 ON t1.order_id = t2.order_id
GROUP BY t1.category;
```

**Category: total sales in 2019 vs 2020** 🔄 `YEAR()` → `strftime`
```sql
SELECT category,
       SUM(CASE WHEN strftime('%Y', order_date) = '2019' THEN sales ELSE 0 END) AS sales_2019,
       SUM(CASE WHEN strftime('%Y', order_date) = '2020' THEN sales ELSE 0 END) AS sales_2020
FROM orders
GROUP BY category;
```

**Top 5 cities in the West region by average days between order and ship date** 🔄 `TOP`→`LIMIT`, `DATEDIFF`→`julianday`
```sql
SELECT city,
       AVG(julianday(ship_date) - julianday(order_date)) AS avg_days
FROM orders
WHERE region = 'West'
GROUP BY city
ORDER BY avg_days DESC
LIMIT 5;
```

**Count occurrences of the character 'n' in customer_name** 🔄 `LEN`→`LENGTH`
```sql
SELECT customer_name,
       LENGTH(customer_name) - LENGTH(REPLACE(LOWER(customer_name), 'n', '')) AS count_of_occurrence_of_n
FROM orders;
```
> 💡 `LOWER()` first so both `'n'` and `'N'` are counted.

**Split customer_name into first_name / last_name (everything after the first space = last name)** 🔄 `LEFT`/`CHARINDEX`/`SUBSTRING` → `SUBSTR`/`INSTR`
```sql
SELECT customer_name,
       SUBSTR(customer_name, 1, INSTR(customer_name, ' ') - 1) AS first_name,
       SUBSTR(customer_name, INSTR(customer_name, ' ') + 1, LENGTH(customer_name)) AS last_name
FROM orders
WHERE INSTR(customer_name, ' ') > 0;
```

**NULLs recap** *(full detail in Chapter 2 §2.1–2.2)*

| Topic | Quick rule |
|---|---|
| Meaning | NULL = unknown/missing, not 0, not blank |
| Arithmetic | Anything + NULL = NULL |
| `IFNULL(expr, replacement)` | SQLite's 2-arg substitute for MSSQL's `ISNULL` |
| `COALESCE(e1, e2, ..., eN)` | Standard, multi-argument, returns first non-NULL |
| `REPLACE(str, NULL, x)` | Returns NULL — `REPLACE` cannot be used to substitute a NULL value, since NULL isn't a string for it to search for |

**CROSS JOIN examples** *(full coverage in Chapter 3 §3.6 — fully portable, no changes)*

**FIREFIGHTERS — workers with the maximum total earnings** *(generic practice table)*
```sql
SELECT COUNT(*) AS "A"
FROM workers
WHERE DailyWage * DaysWorked = (SELECT MAX(DailyWage * DaysWorked) FROM workers);
```
> 💡 `Total earnings = DailyWage × DaysWorked`. The output column is literally named `A` per the original problem spec.

---

## 4.9 Master Function Cheat-Sheet (MSSQL → SQLite, all chapters combined)

| Category | MSSQL | SQLite |
|---|---|---|
| Top N | `TOP N` (after SELECT) | `LIMIT N` (end of query) |
| String concat | `+` | `\|\|` |
| NULL → default | `ISNULL(x, y)` | `IFNULL(x, y)` or `COALESCE(x, y)` |
| First non-null | `COALESCE(...)` | `COALESCE(...)` *(no change)* |
| Year/Month/Day part | `YEAR()/MONTH()/DAY()`, `DATEPART()`, `EXTRACT()` | `strftime('%Y'/'%m'/'%d', date)` |
| Add to date | `DATEADD()` | `DATE(date, '+n unit')` |
| Date difference | `DATEDIFF()` | `julianday(d2) - julianday(d1)` |
| Current date/time | `GETDATE()` | `DATE('now')` / `DATETIME('now')` |
| String length | `LEN()` | `LENGTH()` |
| Substring | `SUBSTRING()` | `SUBSTR()` |
| Left/Right N chars | `LEFT()/RIGHT()` | `SUBSTR(str,1,n)` / `SUBSTR(str,-n)` |
| Find position | `CHARINDEX(needle, haystack)` | `INSTR(haystack, needle)` ⚠️ reversed args |
| Aggregate string join | `STRING_AGG(...) WITHIN GROUP (ORDER BY..)` | `GROUP_CONCAT(col, sep)` (order via subquery or inline `ORDER BY` on ≥3.44) |
| Reverse string | `REVERSE()` | ❌ none — recursive CTE workaround |
| Comparison subquery | `> ANY / > ALL / = ANY` | ❌ none — rewrite with `MIN()`/`MAX()`/`IN` |
| Safe cast | `TRY_CAST` / `TRY_CONVERT` | not needed — plain `CAST` never errors |
| Natural log | `LOG(x)` | `LN(x)` |
| Decimal truncate | `TRUNCATE(x, n)` | `CAST(x * POWER(10,n) AS INT) / POWER(10,n)` |

---

## 4.10 Final Interview Tips ⭐

- ✅ `CASE WHEN` stops at the first TRUE condition — order your conditions from most specific to least specific.
- ✅ `AVG()`/`SUM()`/`COUNT(col)` ignore NULL automatically — you can exploit this by omitting `ELSE` in a `CASE` to effectively filter inside an aggregate.
- ✅ SQLite stores dates as plain ISO-8601 **text** — there is no native DATE type, so `MIN()`/`MAX()`/`ORDER BY` on `'YYYY-MM-DD'` strings sort correctly, but malformed or inconsistent date formats will silently sort wrong.
- ✅ Whenever you see `YEAR()`, `MONTH()`, `DATEDIFF()`, `DATEADD()`, `LEN()`, `CHARINDEX()`, `LEFT()`, `RIGHT()`, `ISNULL()`, `STRING_AGG() WITHIN GROUP`, or `TOP` in old notes — that's your signal to reach for this chapter's conversion table.
- ✅ `ANY`/`ALL` subquery comparisons simply don't exist in SQLite — there's no flag, no pragma, no workaround except rewriting with `MIN`/`MAX`/`IN`.
- ✅ When in doubt about whether a function exists in SQLite, test it — `SELECT function(...)` either runs or throws `no such function`, instantly telling you.

---

🎉 **You've now covered all 4 chapters** — Foundations, NULL/Subqueries, Joins/Sets, and CASE/String/Date functions — fully adapted from MSSQL to SQLite, ready to run in your `mydata.db` notebook.

⭐ *Practice regularly. Understand the logic, not just the syntax. Write efficient queries.* ⭐
