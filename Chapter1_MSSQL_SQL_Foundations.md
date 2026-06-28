# Chapter 1: SQL Foundations — MSSQL Edition
### Definitions, Examples, Code, Interview Points, Tricks, and Practice Questions

This chapter is written for real MSSQL (T-SQL), running in VS Code with the SQL Server extension — not SQLite anymore. So things like `TOP`, `ISNULL`, `CHARINDEX`, `YEAR()`, and full bracket wildcards in `LIKE` all work exactly as written, with no conversion needed.

A note on the example tables used below: `orders` (with columns like `order_id`, `customer_name`, `category`, `region`, `sales`, `profit`, `quantity`, `discount`, `order_date`, `ship_date`, `ship_mode`) and `employee` (with columns like `emp_id`, `emp_name`, `salary`, `dept_id`, `manager_id`). If your actual tables in VS Code use different column names, just swap them in — the logic of every query stays the same.

Since you're preparing for Lead/Senior-level interviews, each topic below includes not just "how," but also the deeper "why" that interviewers usually push for.

---

## 1. SQL 
**Code:**

```sql
-- Wrong — SUM() doesn't exist yet at the WHERE stage
SELECT category, SUM(sales) AS total_sales
FROM orders
WHERE SUM(sales) > 1000
GROUP BY category;
```

```sql
-- Correct — HAVING runs after grouping, so SUM() is available there
SELECT category, SUM(sales) AS total_sales
FROM orders
GROUP BY category
HAVING SUM(sales) > 1000;
```

**Interview Points:**

- Interviewers often ask: "Is this the order the engine *actually* runs things in, or just the order it must *behave as if* it ran them in?"
- Correct answer: it's the second one. This is called the **logical** order. The query optimizer is free to physically run steps in a different sequence (for example, applying a filter before a join) as long as the final answer matches what the logical order guarantees.
- Being able to open the **Execution Plan** (in SSMS or the VS Code SQL Server extension) and explain whether a filter got "pushed down" earlier than expected shows real engine-level understanding, not memorized syntax.

**Tricks / Be Careful:**

- A column alias created in `SELECT` cannot be used inside `WHERE` — because `WHERE` runs before `SELECT` exists.
- An aggregate function (`SUM`, `COUNT`, etc.) cannot be used inside `WHERE` — because grouping hasn't happened yet at that point. Use `HAVING` instead.
- `TOP` is written early but evaluated last — so `TOP` without `ORDER BY` gives you an arbitrary, unpredictable set of rows, not necessarily the "top" anything.

---

## 2. SELECT Basics, and TOP

**Definition:**

- `SELECT *` — shows every column.
- `SELECT DISTINCT city` — shows only unique values, no repeats.
- `SELECT COUNT(*)` — counts every row, even ones with NULLs in them.
- `SELECT COUNT(col)` — counts only rows where that column is NOT NULL.
- `TOP N` — limits the result to N rows. Written right after `SELECT`, before the column list.

**Example:**

- "Give me the 10 most recent orders" needs both a sort (`ORDER BY order_date DESC`) and a limit (`TOP 10`) — `TOP` alone, with no sorting, just grabs 10 rows in whatever order the engine happens to read them in.

**Code:**

```sql
SELECT TOP 10 *
FROM orders
ORDER BY order_date DESC;
```

```sql
-- TOP N PERCENT — returns roughly N% of the total rows, rounded up
SELECT TOP 10 PERCENT *
FROM orders
ORDER BY sales DESC;
```

```sql
-- TOP N WITH TIES — if the cutoff value is tied with the next row, include it too
SELECT TOP 3 WITH TIES *
FROM employee
ORDER BY salary DESC;
```

**Interview Points:**

- Be ready to explain why `SELECT *` is flagged in code review, with real reasons, not just "it's bad style":
  - It blocks the engine from using a **covering index** (an index that already has every column the query needs, so it can skip the table entirely).
  - It silently changes the result's shape if a column gets added to or removed from the table later — breaking anything downstream without any application code changing.

**Tricks / Be Careful:**

- `TOP` without `ORDER BY` is **not deterministic** — running the same query twice can return different rows. Always pair `TOP` with `ORDER BY` when you actually mean "the top N by some measure."
- `TOP N WITH TIES` can return **more** than N rows if there's a tie at the cutoff point — don't assume it always gives you exactly N.
- `TOP N PERCENT` rounds up, so `TOP 10 PERCENT` on 95 rows gives you 10 rows, not 9.5.

---

## 3. WHERE Clause Operators

**Definition and Example for each operator:**

- `=` — checks if two values are exactly equal.
  ```sql
  WHERE region = 'South'
  ```
- `<>` (or `!=`) — checks if two values are NOT equal.
  ```sql
  WHERE region <> 'South'
  ```
- `>`, `<`, `>=`, `<=` — greater than, less than, and their "or equal to" versions.
  ```sql
  WHERE sales > 1000
  ```
- `BETWEEN ... AND` — checks if a value falls inside a range (both ends included).
  ```sql
  WHERE sales BETWEEN 100 AND 500
  ```
- `IN` — checks if a value matches any one value in a list. Shortcut for several `OR` conditions.
  ```sql
  WHERE region IN ('South', 'West')
  ```
- `NOT IN` — checks if a value does NOT match any value in a list.
  ```sql
  WHERE region NOT IN ('South', 'West')
  ```
- `IS NULL` / `IS NOT NULL` — checks whether a value is missing/empty. In SQL, missing data is called NULL — it is not the same as zero or an empty string.
  ```sql
  WHERE city IS NULL
  ```
- `EXISTS` — checks if a smaller query (a subquery) returns at least one row.
  ```sql
  WHERE EXISTS (SELECT 1 FROM returns WHERE returns.order_id = orders.order_id)
  ```

**Code (combined example):**

```sql
SELECT *
FROM orders
WHERE region = 'South'
  AND discount > 0;
```

**Interview Points:**

- Know the term **sargable** by name — this term actually comes from SQL Server's own documentation history, so it fits naturally in an MSSQL interview.
- A condition is sargable when the engine can use an index to jump straight to matching rows (an index seek), instead of checking every row one at a time (a full scan).
- Plain comparisons like `sales > 1000` or `order_date = '2020-12-01'` are sargable.
- Wrapping the column in a function — like `YEAR(order_date) = 2020` — is **not** sargable, because the engine has to compute `YEAR()` on every single row before it can compare anything. This silently disables the index even if one exists on `order_date`.
- The fix: rewrite it as a range on the raw column instead:
  ```sql
  WHERE order_date >= '2020-01-01' AND order_date < '2021-01-01'
  ```

**Tricks / Be Careful:**

- Always use single quotes for text values — `'South'`, not `"South"`. By default, MSSQL treats double quotes as identifiers (column/table names), controlled by a setting called `QUOTED_IDENTIFIER`. If that setting is OFF, double quotes can behave like strings instead — which makes your code behave differently depending on a session setting nobody else can see. Stick to single quotes always.
- The **`NOT IN` + NULL trap** (this happens in every database, not just MSSQL): if the list or subquery feeding `NOT IN` contains even one NULL, the whole condition becomes `UNKNOWN` for every row, and the query silently returns zero rows.
  ```sql
  -- If returns.order_id can ever be NULL, this silently returns nothing
  -- even when unreturned orders genuinely exist
  SELECT *
  FROM orders
  WHERE order_id NOT IN (SELECT order_id FROM returns);
  ```
- Whether `=` and `LIKE` are case-sensitive depends on the **collation** set on your database/column — not on the SQL standard. Most default SQL Server installations use a case-insensitive collation, but this isn't guaranteed everywhere, so don't assume it blindly.

---

## 4. AND / OR / NOT Precedence

**Definition:**

- `AND` — both conditions must be true.
- `OR` — at least one condition must be true.
- `NOT` — flips true to false and false to true.
- SQL checks `NOT` first, then `AND`, then `OR`. So `A OR B AND C` is actually read as `A OR (B AND C)`, not `(A OR B) AND C`.

**Example:**

- Goal: orders that are (Technology OR Furniture) AND only from the year 2020.

**Code:**

```sql
-- Looks correct, but is actually wrong
SELECT *
FROM orders
WHERE category = 'Technology' OR category = 'Furniture'
  AND YEAR(order_date) = 2020;
```

- Because `AND` binds tighter, this is really read as:
  `category = 'Technology' OR (category = 'Furniture' AND YEAR(order_date) = 2020)`
- Result: every Technology order leaks in, no matter the year. Only Furniture gets restricted to 2020.

```sql
-- Correct — brackets force the intended grouping
SELECT *
FROM orders
WHERE category IN ('Technology', 'Furniture')
  AND YEAR(order_date) = 2020;
```

**Interview Points:**

- This exact bug shows up most often in dynamically built SQL — ORM query builders, or filters assembled conditionally in application code — where nobody is reading the final SQL text directly.
- The senior-level habit: always add your own brackets around mixed `AND`/`OR` conditions, even when you're sure about precedence rules. It protects you and whoever reviews the code next.

**Tricks / Be Careful:**

- `YEAR(order_date) = 2020` is not sargable (see the sargability point in section 3) — it's correct logic, but not the fastest way to write it on a large, indexed table. A range filter on the raw date column would be faster.
- Never assume a reviewer (or your future self) will mentally re-derive precedence correctly under time pressure — bracket explicitly.

---

## 5. LIKE and Wildcards

**Definition:**

- A wildcard is a symbol that stands in for "any character" or "any text," so you can match patterns instead of exact text.
- MSSQL's `LIKE` supports four wildcard types:
  - `%` — any number of characters (including zero)
  - `_` — exactly one character
  - `[set]` — any one character from the set, e.g. `[SM]` means S or M
  - `[^set]` — any one character NOT in the set
  - `[a-m]` — any one character in a range

**Example:**

- "Names starting with S or M" needs a character-set wildcard, not just `%` or `_`.

**Code:**

```sql
SELECT * FROM orders WHERE customer_name LIKE 'S%';        -- starts with S
SELECT * FROM orders WHERE customer_name LIKE '%an%';       -- contains "an" anywhere
SELECT * FROM orders WHERE customer_name LIKE '_____';      -- exactly 5 characters
SELECT * FROM orders WHERE customer_name LIKE '[SM]%';      -- starts with S or M
SELECT * FROM orders WHERE customer_name LIKE '[^SM]%';     -- does NOT start with S or M
SELECT * FROM orders WHERE customer_name LIKE '[A-M]%';     -- starts with a letter A through M
```

Worked example — second letter is a vowel, third letter is NOT a vowel:

```sql
SELECT customer_name
FROM orders
WHERE customer_name LIKE '_[aeiou][^aeiou]%';
```

Worked example — name doesn't start with A and doesn't end with n (needs two separate conditions, not one combined pattern):

```sql
SELECT *
FROM orders
WHERE customer_name NOT LIKE 'A%'
  AND customer_name NOT LIKE '%n';
```

Worked example — name contains exactly two letter a's:

```sql
SELECT customer_name
FROM orders
WHERE LEN(customer_name) - LEN(REPLACE(customer_name, 'a', '')) = 2;
```

- `LIKE '%a%a%'` would be wrong here — that means "at least two a's somewhere," not "exactly two."

**Interview Points:**

- This applies to every database engine, not just MSSQL: a wildcard at the **start** of the pattern (`LIKE '%an%'`) can never use a standard index, because the index is sorted from the first character, and the engine has no way to know where a string containing "an" in the middle would land in that sorted order.
- A wildcard only at the **end** (`LIKE 'San%'`) can use the index, because every matching value sits in one contiguous range starting with "San."
- This is one of the most common "why is this query slow" questions asked at senior level.

**Tricks / Be Careful:**

- If you need to search for a literal `%`, `_`, or `[` character (not as a wildcard), wrap it in square brackets: `LIKE '%[%]%'` matches values that contain an actual percent sign. There's also a formal `ESCAPE` clause for the same purpose.
- Case sensitivity inside `LIKE` depends on the column's/database's collation setting — not a fixed rule. Don't assume it behaves the same way on every server.

---

## 6. ORDER BY

**Definition:**

- Sorts the final result.
- Default direction is ascending (`ASC`) if you don't specify `ASC` or `DESC`.
- Multiple sort columns are applied left to right — ties on the first column are broken by the second, and so on.

**Example:**

- "Sort orders by city, and within each city, show the highest sales first."

**Code:**

```sql
SELECT * FROM orders ORDER BY sales DESC;
SELECT * FROM orders ORDER BY city ASC, sales DESC;
```

**Interview Points:**

- `ORDER BY` on a large result forces a full sort, unless the engine can pull the rows directly from an index that's already stored in that order. The Execution Plan will show a `Sort` operator if a real sort is happening — worth checking if a query feels slower than its filtering alone would explain.
- Pairing `ORDER BY` with `TOP` doesn't remove the sort, but it does let the engine stop early once it has enough rows — part of why this combination is the right way to get "top N" results (see section 1 and 2).

**Tricks / Be Careful:**

- Don't rely on a query returning rows in any particular order unless you wrote an explicit `ORDER BY` for it — the engine makes no promises otherwise, even if it "looks" sorted today.

---

## 7. Aggregate Functions

**Definition:**

- `COUNT(*)` — counts every row, NULLs included.
- `COUNT(col)` — counts only rows where `col` is NOT NULL.
- `COUNT(DISTINCT col)` — counts unique non-NULL values.
- `SUM`, `AVG`, `MIN`, `MAX` — all ignore NULL values in the column they're working on.

**Example:**

- If `discount` has some NULLs, `COUNT(*)` and `COUNT(discount)` will give different numbers.

**Code:**

```sql
SELECT
    COUNT(*)         AS total_rows,
    COUNT(discount)   AS rows_with_discount,
    SUM(sales)        AS total_sales,
    AVG(sales)        AS avg_sales,
    MIN(profit)       AS min_profit,
    MAX(profit)       AS max_profit
FROM orders;
```

```sql
-- AVG can also be computed manually this way
SELECT SUM(sales) / COUNT(sales) AS avg_sales   -- COUNT(sales), not COUNT(*), to match AVG's NULL handling
FROM orders;
```

**Interview Points:**

- Common trick question: "Is `COUNT(1)` faster than `COUNT(*)`?" Answer: no — in every modern optimizer, including SQL Server's, they compile to the exact same plan, since neither one actually evaluates any column. The real difference that matters is `COUNT(*)`/`COUNT(1)` versus `COUNT(column)` — the column version genuinely does more work, checking each row for NULL before counting it.

**Tricks / Be Careful:**

- `SUM`, `AVG`, `MIN`, `MAX`, and `COUNT(col)` all ignore NULLs.
- `DISTINCT` does **not** ignore NULLs the same way — it treats all the NULLs as one single value and keeps just one row for them in the output. So `SELECT DISTINCT city` will show one row for NULL if any city values are missing, rather than dropping NULLs entirely.

---

## 8. GROUP BY and HAVING

**Definition:**

- `GROUP BY` — puts rows into groups based on column values.
- `HAVING` — filters out whole groups after the grouping/aggregation has already happened.
- `WHERE` filters rows before grouping; `HAVING` filters groups after grouping. This is the only place an aggregate function is allowed as a filter condition.

**Example:**

- "Show categories where total sales are over 10,000" needs `HAVING`, because the filter condition (`SUM(sales) > 10000`) depends on an aggregate.

**Code:**

```sql
SELECT category, SUM(sales) AS total_sales
FROM orders
GROUP BY category
HAVING SUM(sales) > 10000;
```

**Interview Points:**

- There's a real performance reason to prefer `WHERE` over `HAVING` whenever a condition doesn't actually depend on an aggregate. `WHERE` throws away non-matching rows **before** the expensive grouping work happens; `HAVING` only gets to filter **after** every group has already been fully built.
- Example: in a query that needs both "only Furniture" and "total sales over 10,000," the Furniture filter belongs in `WHERE`, not `HAVING` — even though writing it as `HAVING category = 'Furniture'` would still give the correct answer, it forces the engine to aggregate every category first and throw most of them away afterward.
- Spotting this kind of misplaced filter in someone else's query, and explaining *why* it's slower (not just that it "should" be in WHERE), is a realistic Lead-level code review answer.

**Tricks / Be Careful:**

- `HAVING` can actually be used **without** `GROUP BY` — it just treats the entire table as a single group. Not common, but a good thing to know exists:
  ```sql
  SELECT SUM(sales) AS total_sales
  FROM orders
  HAVING SUM(sales) > 100000;
  ```

---

## 9. DISTINCT vs GROUP BY

**Definition:**

- `DISTINCT` — removes duplicate rows, no aggregation involved.
- `GROUP BY` — forms groups, almost always paired with an aggregate function, and is more powerful when you need per-group calculations rather than just uniqueness.

**Example:**

- "List the unique cities we ship to" — use `DISTINCT`.
- "List each city along with its total sales" — use `GROUP BY`.

**Code:**

```sql
SELECT DISTINCT city FROM orders;

SELECT category, COUNT(*) AS order_count
FROM orders
GROUP BY category;
```

**Interview Points:**

- It's commonly believed that `DISTINCT` is slower than an equivalent `GROUP BY`. In most modern optimizers, including SQL Server's, the two compile to the same plan when they express the same intent, because removing duplicates and forming groups both need the engine to sort or hash the same columns.
- Rather than answering from memory, the senior-level move is to check directly using the Execution Plan (in SSMS: "Display Estimated Execution Plan"; in VS Code's SQL Server extension, the equivalent plan view) and compare the two plans rather than repeating a rule of thumb.

**Tricks / Be Careful:**

- Remember the NULL behavior difference from section 7: `DISTINCT` keeps a single row for NULL rather than dropping it, which can be surprising if you expected NULL to be excluded the way it is from `COUNT`, `SUM`, `AVG`, `MIN`, and `MAX`.

---

## Practice Questions (QnA)

Use these to test yourself after going through the chapter. Try writing the query yourself before checking the answer.

**Q1.** Get all orders where the customer's name has "a" as the second character and "d" as the fourth character.

```sql
SELECT *
FROM orders
WHERE customer_name LIKE '_a_d%';
```

**Q2.** Get all orders placed in the month of December 2020.

```sql
SELECT *
FROM orders
WHERE MONTH(order_date) = 12
  AND YEAR(order_date) = 2020;
```

**Q3.** Get all orders where ship_mode is neither 'Standard Class' nor 'First Class' and ship_date is after November 2020.

```sql
SELECT *
FROM orders
WHERE ship_mode NOT IN ('Standard Class', 'First Class')
  AND ship_date >= '2020-12-01';
```

**Q4.** Get all orders where the customer name neither starts with "A" nor ends with "n".

```sql
SELECT *
FROM orders
WHERE customer_name NOT LIKE 'A%'
  AND customer_name NOT LIKE '%n';
```

**Q5.** Get all orders where profit is negative.

```sql
SELECT * FROM orders WHERE profit < 0;
```

**Q6.** Get all orders where either quantity is less than 3 or profit is 0.

```sql
SELECT * FROM orders WHERE quantity < 3 OR profit = 0;
```

**Q7.** Get all orders from the South region where some discount is provided to the customer.

```sql
SELECT * FROM orders WHERE region = 'South' AND discount > 0;
```

**Q8.** Find the top 5 orders with the highest sales in the Furniture category.

```sql
SELECT TOP 5 order_id, SUM(sales) AS total_sales_per_order_id
FROM orders
WHERE category = 'Furniture'
GROUP BY order_id
ORDER BY total_sales_per_order_id DESC;
```

**Q9.** Find all the records in the Technology and Furniture categories for orders placed in the year 2020 only. (Watch the precedence trap from section 4.)

```sql
SELECT *
FROM orders
WHERE category IN ('Technology', 'Furniture')
  AND YEAR(order_date) = 2020;
```

**Q10.** Find all the orders where order date is in year 2020 but ship date is in year 2021.

```sql
SELECT *
FROM orders
WHERE YEAR(order_date) = 2020
  AND YEAR(ship_date) = 2021;
```

**Q11.** Find employees whose salary is greater than the average salary of their own department.

```sql
SELECT *
FROM employee t1
WHERE t1.salary > (
    SELECT AVG(salary)
    FROM employee t2
    WHERE t1.dept_id = t2.dept_id
);
```

**Q12.** Find duplicate employee names in a table.

```sql
SELECT emp_name, COUNT(*) AS name_count
FROM employee
GROUP BY emp_name
HAVING COUNT(*) > 1;
```

**Q13.** Find Los Angeles orders where the order amount is greater than the city's average order amount.

```sql
SELECT *
FROM orders
WHERE city = 'Los Angeles'
  AND sales > (
      SELECT AVG(sales) FROM orders WHERE city = 'Los Angeles'
  );
```

**Q14.** Find the top 3 highest-paid employees.

```sql
SELECT TOP 3 *
FROM employee
ORDER BY salary DESC;
```

If duplicate salaries need to be handled so you get 3 genuinely distinct salary values:

```sql
SELECT TOP 3 DISTINCT salary
FROM employee
ORDER BY salary DESC;
```

**Q15.** Find cities where total FD (fixed deposit) amount is greater than 10 lakh.

```sql
SELECT city, SUM(fd_amount) AS total_fd
FROM fd_customers
GROUP BY city
HAVING SUM(fd_amount) > 1000000;
```

**Q16.** Use a subquery directly inside the SELECT list to show each customer's total sales alongside their name.

```sql
SELECT
    t1.customer_name,
    (SELECT SUM(sales) FROM orders t2 WHERE t1.customer_id = t2.customer_id) AS total_sales
FROM customers t1;
```

**Q17.** Use `IN` with a subquery to find all orders placed by customers based in Mumbai.

```sql
SELECT *
FROM orders
WHERE customer_id IN (
    SELECT customer_id FROM customers WHERE city = 'Mumbai'
);
```

**Q18.** Use `EXISTS` with a correlated subquery to find all orders that have at least one matching return.

```sql
SELECT *
FROM orders o
WHERE EXISTS (
    SELECT 1 FROM returns r WHERE r.order_id = o.order_id
);
```

---

Next: Chapter 2 — NULL Handling, Subqueries and Comparison Operators.
