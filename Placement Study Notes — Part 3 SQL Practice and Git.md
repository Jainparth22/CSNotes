# Placement Study Notes — Part 3: SQL Practice and Git

*Companion to Part 1 (DSA) and Part 2 (OS, DBMS, Networks). Theory behind indexes, ACID and isolation is in Part 2 (B2, B6, B7, B10); this part is the hands-on query and workflow side.*

## 0. How to use these notes

Every topic has: **idea**, **syntax**, **worked example with the output you should expect**, **interview traps**. For SQL, run every example yourself on the sample data in section S0 (PostgreSQL, MySQL 8 and SQLite all accept almost everything here; dialect differences are flagged).

**Contents:** Part S (SQL): S0 sample schema · S1 query basics and DML/DDL · S2 joins and NULL handling · S3 subqueries, CTEs, aggregates, CASE, UNION, EXISTS/IN · S4 window functions and classic interview problems · S5 database design and performance in SQL (views, procedures, indexes, EXPLAIN, transactions) · S6 practice set with answers. Part G (Git): G1 mental model · G2 everyday commands · G3 undo toolbox (restore, reset, revert, reflog) · G4 branching, merge vs rebase, conflicts · G5 stash, cherry-pick, bisect, tags · G6 team workflow, PRs, branching strategies, `.gitignore` · G7 GitHub Actions CI/CD · G8 scenario Q&A and cheat sheet.

---

# PART S: SQL

## S0. Sample schema used in all examples

```sql
CREATE TABLE departments (
  dept_id   INT PRIMARY KEY,
  dept_name VARCHAR(30) NOT NULL
);
CREATE TABLE employees (
  emp_id     INT PRIMARY KEY,
  name       VARCHAR(30) NOT NULL,
  salary     INT,
  dept_id    INT REFERENCES departments(dept_id),
  manager_id INT REFERENCES employees(emp_id),
  hire_date  DATE
);
INSERT INTO departments VALUES (1,'Engineering'),(2,'Sales'),(3,'HR'),(4,'Legal');
INSERT INTO employees VALUES
 (1,'Asha', 90000, 1, NULL, '2019-01-10'),
 (2,'Ravi', 70000, 1, 1,    '2020-03-15'),
 (3,'Meera',70000, 1, 1,    '2021-07-01'),
 (4,'Kiran',60000, 2, 1,    '2018-11-20'),
 (5,'Sana', 85000, 2, 4,    '2019-06-30'),
 (6,'Dev',  50000, 2, 4,    '2022-02-14'),
 (7,'Neha', 95000, 3, 1,    '2017-09-09'),
 (8,'Omar', 40000, NULL, 7, '2023-01-05');   -- Omar has no department; Legal has no employees

CREATE TABLE orders (
  order_id    INT PRIMARY KEY,
  customer_id INT,
  order_date  DATE,
  amount      DECIMAL(10,2),
  status      VARCHAR(10)
);
INSERT INTO orders VALUES
 (1,101,'2026-01-01',500,'PAID'), (2,101,'2026-01-02',300,'PAID'),
 (3,102,'2026-01-02',700,'CANCELLED'), (4,103,'2026-01-03',200,'PAID'),
 (5,101,'2026-01-05',100,'PAID'), (6,102,'2026-01-06',900,'PAID');
```

---

## S1. Query basics, DML and DDL

**Logical execution order (this explains most errors):** `FROM/JOIN` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `DISTINCT` → `ORDER BY` → `LIMIT/OFFSET`. Writing order is `SELECT ... FROM ... WHERE ... GROUP BY ... HAVING ... ORDER BY ... LIMIT`.

```sql
-- SELECT, WHERE, ORDER BY, LIMIT
SELECT name, salary
FROM employees
WHERE salary >= 70000 AND dept_id IS NOT NULL      -- IS NULL / IS NOT NULL, never = NULL
ORDER BY salary DESC, name ASC                      -- multi-key sort
LIMIT 3;                                            -- SQL Server: SELECT TOP 3 ...; Oracle/standard: FETCH FIRST 3 ROWS ONLY
-- Output: Neha 95000, Asha 90000, Sana 85000

-- Pattern filters
WHERE name LIKE 'A%'            -- starts with A  (% any length, _ one character)
WHERE salary BETWEEN 50000 AND 70000   -- inclusive on both ends
WHERE dept_id IN (1, 2)
WHERE hire_date >= '2020-01-01' AND hire_date < '2021-01-01'   -- sargable date range
```

**GROUP BY and HAVING**

```sql
SELECT dept_id, COUNT(*) AS headcount, AVG(salary) AS avg_sal, MAX(salary) AS top_sal
FROM employees
GROUP BY dept_id
HAVING COUNT(*) >= 3;                -- HAVING filters GROUPS; WHERE filters ROWS before grouping
-- Output: dept 1 -> 3, 76666.67, 90000;  dept 2 -> 3, 65000, 85000
```

**Rule:** every non-aggregated column in `SELECT` must be in `GROUP BY` (MySQL without `ONLY_FULL_GROUP_BY` silently picks an arbitrary value: a bug source).

**DML**

```sql
INSERT INTO employees (emp_id, name, salary, dept_id) VALUES (9, 'Isha', 65000, 1);
INSERT INTO employees_backup SELECT * FROM employees WHERE dept_id = 1;     -- insert from a query
UPDATE employees SET salary = salary * 1.10 WHERE dept_id = 2;              -- ALWAYS include WHERE
DELETE FROM employees WHERE emp_id = 9;
-- Upsert: PostgreSQL ON CONFLICT (emp_id) DO UPDATE SET ...; MySQL ON DUPLICATE KEY UPDATE ...
```

**DDL**

```sql
CREATE TABLE project (id INT PRIMARY KEY, title VARCHAR(50) NOT NULL);
ALTER TABLE project ADD COLUMN budget DECIMAL(12,2) DEFAULT 0;
ALTER TABLE project DROP COLUMN budget;
ALTER TABLE project RENAME TO projects;
DROP TABLE projects;                      -- removes structure and data
TRUNCATE TABLE employees_backup;          -- removes all rows, keeps structure
```

**DELETE vs TRUNCATE vs DROP**

|  | DELETE | TRUNCATE | DROP |
| --- | --- | --- | --- |
| Category | DML | DDL (treated like DDL) | DDL |
| Removes | chosen rows (with `WHERE`) | **all rows** | the whole table (structure, data, indexes, privileges) |
| `WHERE` allowed | yes | no | no |
| Speed | slowest (logs each row) | fast (deallocates pages) | fast |
| Rollback | yes | PostgreSQL/SQL Server: yes inside a transaction; **MySQL/Oracle: no** (implicit commit) | MySQL/Oracle: no |
| Triggers fire | yes (row triggers) | no | no |
| Auto-increment reset | no | **yes** | n/a |
| Foreign keys | checked per row | blocked if referenced by another table's FK | blocked if referenced |

**Traps**

- `UPDATE`/`DELETE` without `WHERE` hits every row; run the `WHERE` as a `SELECT` first, and use a transaction.
- `WHERE salary > 50000 AND dept_id = 1 OR dept_id = 2` mixes precedence (`AND` binds tighter than `OR`); use parentheses.
- Aliases from `SELECT` cannot be used in `WHERE` (but can in `ORDER BY`, and in `GROUP BY` in some dialects).
- `LIMIT` without `ORDER BY` returns arbitrary rows.
- `COUNT(*)` counts rows; `COUNT(col)` skips NULLs; `COUNT(DISTINCT col)` counts unique non-NULL values; `AVG` and `SUM` ignore NULLs (and `SUM` of no rows is NULL, not 0: use `COALESCE(SUM(x), 0)`).
- Integer division: `5 / 2 = 2` in many dialects; use `5 * 1.0 / 2` or cast.
- `DISTINCT` applies to the whole selected row, not just the first column.

---

## S2. Joins, self joins and NULL handling

```
A: dept 1,2,3,4                 B (employees dept_id): 1,1,1,2,2,2,3,NULL
INNER   = rows with a match on both sides
LEFT    = every row of the left table + matches (NULL where no match)
RIGHT   = mirror of LEFT
FULL    = LEFT union RIGHT (all rows from both)   (MySQL has no FULL JOIN: emulate with LEFT UNION RIGHT)
CROSS   = every pair (n * m rows)
```

```sql
-- INNER JOIN: only employees that have a department (Omar disappears, Legal disappears)
SELECT e.name, d.dept_name
FROM employees e
INNER JOIN departments d ON d.dept_id = e.dept_id;

-- LEFT JOIN: departments with their employees, Legal appears with NULL name
SELECT d.dept_name, e.name
FROM departments d LEFT JOIN employees e ON e.dept_id = d.dept_id;

-- Anti-join: departments with NO employees (Legal)
SELECT d.dept_name
FROM departments d LEFT JOIN employees e ON e.dept_id = d.dept_id
WHERE e.emp_id IS NULL;

-- FULL JOIN: shows both Omar (no dept) and Legal (no employees)
SELECT e.name, d.dept_name
FROM employees e FULL OUTER JOIN departments d ON d.dept_id = e.dept_id;

-- Counting per group INCLUDING groups with zero: LEFT JOIN + COUNT(child column), not COUNT(*)
SELECT d.dept_name, COUNT(e.emp_id) AS headcount
FROM departments d LEFT JOIN employees e ON e.dept_id = d.dept_id
GROUP BY d.dept_name;          -- Legal -> 0 (COUNT(*) would wrongly give 1)
```

**Self join (a table joined to itself via aliases)**

```sql
-- Employee and their manager
SELECT e.name AS employee, m.name AS manager
FROM employees e LEFT JOIN employees m ON m.emp_id = e.manager_id;     -- LEFT so the CEO (Asha) still appears

-- Employees earning more than their manager
SELECT e.name FROM employees e JOIN employees m ON m.emp_id = e.manager_id
WHERE e.salary > m.salary;

-- Pairs of employees in the same department with the same salary (avoid duplicates and self-pairs with <)
SELECT a.name, b.name FROM employees a JOIN employees b
  ON a.dept_id = b.dept_id AND a.salary = b.salary AND a.emp_id < b.emp_id;       -- Ravi, Meera
```

**Multi-column joins:** `ON a.x = b.x AND a.y = b.y` (composite keys). **Non-equi joins:** `ON t.value BETWEEN r.low AND r.high` (range lookups).

**ON vs WHERE in outer joins (a top interview trap)**

```sql
-- Intended: every department, with only its high earners
SELECT d.dept_name, e.name
FROM departments d LEFT JOIN employees e ON e.dept_id = d.dept_id AND e.salary > 80000;   -- filter in ON: keeps all departments
-- Putting AND e.salary > 80000 in WHERE would drop unmatched departments (NULL fails the condition) and turn the LEFT JOIN into an INNER JOIN.
```

**NULL handling (three-valued logic: TRUE, FALSE, UNKNOWN)**

- Any comparison with NULL is UNKNOWN: `NULL = NULL` is not true. Use `IS NULL`, `IS NOT NULL`, or `IS NOT DISTINCT FROM` (PostgreSQL) / `<=>` (MySQL) for null-safe equality.
- `WHERE` keeps only rows where the condition is TRUE, so `salary <> 70000` also drops NULL salaries.
- Joins on NULL keys never match: Omar's NULL `dept_id` joins nothing.
- `COALESCE(x, 0)` returns the first non-NULL; `NULLIF(a, b)` returns NULL if equal (avoid divide by zero: `x / NULLIF(y, 0)`).
- `GROUP BY` puts all NULLs in one group; `DISTINCT` and `UNION` treat NULLs as equal; `ORDER BY` places NULLs first or last depending on the engine (`NULLS FIRST/LAST` in PostgreSQL/Oracle).
- **`NOT IN` with a NULL in the list returns no rows:** `x NOT IN (1, 2, NULL)` is UNKNOWN for every x not equal to 1 or 2. Use `NOT EXISTS`.

**Join multiplication trap:** joining a one-to-many relationship and then `SUM`ming a parent column inflates totals (each parent row is repeated per child). Aggregate each side in a subquery/CTE before joining.

---

## S3. Subqueries, CTEs, aggregates, CASE, UNION, EXISTS vs IN

**Subquery kinds**

```sql
-- Scalar subquery (one value)
SELECT name FROM employees WHERE salary > (SELECT AVG(salary) FROM employees);

-- IN subquery (list)
SELECT name FROM employees WHERE dept_id IN (SELECT dept_id FROM departments WHERE dept_name IN ('Sales','HR'));

-- Derived table (subquery in FROM) must have an alias
SELECT t.dept_id, t.avg_sal FROM (SELECT dept_id, AVG(salary) avg_sal FROM employees GROUP BY dept_id) t WHERE t.avg_sal > 60000;

-- Correlated subquery: references the outer row, re-evaluated per outer row
SELECT e.name, e.salary FROM employees e
WHERE e.salary > (SELECT AVG(salary) FROM employees WHERE dept_id = e.dept_id);   -- above their own department average
```

A correlated subquery is conceptually O(n \* m); optimizers often rewrite it as a join, and window functions (S4) are usually faster.

**EXISTS vs IN; NOT EXISTS and anti-joins**

```sql
-- Departments that have at least one employee
SELECT dept_name FROM departments d WHERE EXISTS (SELECT 1 FROM employees e WHERE e.dept_id = d.dept_id);
-- Departments with no employee (three equivalent anti-join forms)
SELECT dept_name FROM departments d WHERE NOT EXISTS (SELECT 1 FROM employees e WHERE e.dept_id = d.dept_id);   -- safest
SELECT dept_name FROM departments WHERE dept_id NOT IN (SELECT dept_id FROM employees);                          -- BROKEN: Omar's NULL dept_id => returns NOTHING
SELECT d.dept_name FROM departments d LEFT JOIN employees e ON e.dept_id = d.dept_id WHERE e.emp_id IS NULL;     -- join form
```

|  | `EXISTS` | `IN` |
| --- | --- | --- |
| Mechanism | stops at the first match (semi-join) | compares against a list/set |
| NULL behaviour | unaffected by NULLs in the subquery | `NOT IN` breaks if the list has a NULL |
| Typical use | correlated existence checks, large subquery | small static lists |
| Performance | modern optimizers treat them similarly; test with EXPLAIN |  |

**CTEs (`WITH`)**

```sql
WITH dept_avg AS (
  SELECT dept_id, AVG(salary) AS avg_sal FROM employees GROUP BY dept_id
)
SELECT e.name, e.salary, d.avg_sal
FROM employees e JOIN dept_avg d ON d.dept_id = e.dept_id
WHERE e.salary > d.avg_sal;

-- Recursive CTE: org chart / hierarchy (anchor + recursive part)
WITH RECURSIVE chain AS (
  SELECT emp_id, name, manager_id, 1 AS lvl FROM employees WHERE manager_id IS NULL       -- anchor: the CEO
  UNION ALL
  SELECT e.emp_id, e.name, e.manager_id, c.lvl + 1
  FROM employees e JOIN chain c ON e.manager_id = c.emp_id                                -- recursive step
)
SELECT * FROM chain ORDER BY lvl;
-- Also generates series: WITH RECURSIVE n(i) AS (SELECT 1 UNION ALL SELECT i + 1 FROM n WHERE i < 10) SELECT i FROM n;
```

CTEs improve readability and can be reused within one statement. Older PostgreSQL (before 12) treated them as optimization fences; recursive CTEs can loop forever if the hierarchy has cycles (add a depth limit).

**Aggregates:** `COUNT`, `SUM`, `AVG`, `MIN`, `MAX` (+ `STRING_AGG`/`GROUP_CONCAT`, `ARRAY_AGG`). `HAVING` filters groups. `GROUP BY ROLLUP/CUBE` produce subtotals.

**CASE WHEN and conditional aggregation**

```sql
SELECT name, salary,
  CASE WHEN salary >= 90000 THEN 'High' WHEN salary >= 60000 THEN 'Mid' ELSE 'Low' END AS band
FROM employees;

-- Conditional aggregation: pivot-style counts in a single scan
SELECT customer_id,
  SUM(CASE WHEN status = 'PAID' THEN amount ELSE 0 END) AS paid_total,
  COUNT(CASE WHEN status = 'CANCELLED' THEN 1 END)      AS cancelled_orders      -- COUNT ignores the NULLs from the missing ELSE
FROM orders GROUP BY customer_id;
-- 101 -> 900, 0;  102 -> 900, 1;  103 -> 200, 0

-- Percentage
SELECT ROUND(100.0 * SUM(CASE WHEN status='PAID' THEN 1 ELSE 0 END) / COUNT(*), 1) AS paid_pct FROM orders;
```

**UNION vs UNION ALL:** `UNION` removes duplicates (an extra sort or hash step); `UNION ALL` keeps everything and is faster; both need the same number of columns with compatible types. Also `INTERSECT`, `EXCEPT` (`MINUS` in Oracle).

**Interview traps:** `NOT IN` + NULL; `SELECT *` with joins duplicating column names; correlated subqueries that scan per row (rewrite with a join or window function); `HAVING` used where `WHERE` would filter earlier and cheaper; a scalar subquery returning more than one row raises an error; `UNION` when `UNION ALL` is intended.

---

## S4. Window functions and classic interview problems

**Idea:** a window function computes a value for **each row** using a set of related rows (the window) **without collapsing rows** the way `GROUP BY` does.

```
function() OVER ( PARTITION BY col      -- split into groups (optional)
                  ORDER BY col          -- order inside each group (needed for ranking, LAG/LEAD, running totals)
                  ROWS BETWEEN ... )    -- the frame (optional)
```

Window functions run **after** `WHERE`/`GROUP BY`/`HAVING`, so you cannot filter on them in the same `SELECT`'s `WHERE`; wrap them in a subquery or CTE.

### Ranking functions: ROW\_NUMBER vs RANK vs DENSE\_RANK

```sql
SELECT name, salary,
  ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn,
  RANK()       OVER (ORDER BY salary DESC) AS rnk,
  DENSE_RANK() OVER (ORDER BY salary DESC) AS drnk
FROM employees;
```

| name | salary | ROW\_NUMBER | RANK | DENSE\_RANK |
| --- | --- | --- | --- | --- |
| Neha | 95000 | 1 | 1 | 1 |
| Asha | 90000 | 2 | 2 | 2 |
| Sana | 85000 | 3 | 3 | 3 |
| Ravi | 70000 | 4 | 4 | 4 |
| Meera | 70000 | 5 | 4 | 4 |
| Kiran | 60000 | 6 | 6 | 5 |
| Dev | 50000 | 7 | 7 | 6 |
| Omar | 40000 | 8 | 8 | 7 |

- **ROW\_NUMBER:** unique sequence, ties broken arbitrarily (add a tie-breaker column for determinism).
- **RANK:** ties share a rank and **leave gaps** (4, 4, 6).
- **DENSE\_RANK:** ties share a rank, **no gaps** (4, 4, 5).
- `NTILE(n)` splits rows into n buckets; `PERCENT_RANK`, `CUME_DIST` give relative position.

### Nth highest salary

```sql
-- Robust: DENSE_RANK handles ties (distinct salary levels)
SELECT DISTINCT salary FROM (
  SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS r FROM employees
) t WHERE r = 2;                                   -- 90000 (change 2 to N)

-- Without window functions (second highest)
SELECT MAX(salary) FROM employees WHERE salary < (SELECT MAX(salary) FROM employees);

-- LIMIT/OFFSET: must use DISTINCT and wrap so it returns NULL (not no rows) when absent
SELECT (SELECT DISTINCT salary FROM employees ORDER BY salary DESC LIMIT 1 OFFSET 1) AS second_highest;
```

**Trap:** `ORDER BY salary DESC LIMIT 1 OFFSET 1` without `DISTINCT` returns the repeated top salary instead of the second distinct value when the highest salary is shared; `RANK` skips numbers, so `RANK() = 2` can return nothing when two people tie for first.

### Top-N per group

```sql
-- Top 2 salary levels per department (ties included)
SELECT dept_id, name, salary FROM (
  SELECT dept_id, name, salary,
         DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS r
  FROM employees WHERE dept_id IS NOT NULL
) t WHERE r <= 2;
-- Engineering: Asha, Ravi, Meera (tie);  Sales: Sana, Kiran;  HR: Neha

-- Highest paid employee per department (use ROW_NUMBER for exactly one row; add a tie-breaker)
SELECT * FROM (SELECT *, ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC, emp_id) AS rn FROM employees) t WHERE rn = 1;
```

### LAG / LEAD, running totals and moving averages

```sql
-- Compare each order with the customer's previous order
SELECT customer_id, order_date, amount,
  LAG(amount)  OVER (PARTITION BY customer_id ORDER BY order_date) AS prev_amount,      -- NULL for the first row
  LEAD(amount) OVER (PARTITION BY customer_id ORDER BY order_date) AS next_amount,
  amount - LAG(amount, 1, 0) OVER (PARTITION BY customer_id ORDER BY order_date) AS change   -- 3rd arg = default
FROM orders;
-- customer 101: (500, NULL, 300), (300, 500, 100), (100, 300, NULL)

-- Running total per customer
SELECT customer_id, order_date, amount,
  SUM(amount) OVER (PARTITION BY customer_id ORDER BY order_date
                    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM orders;                                       -- customer 101: 500, 800, 900

-- 3-row moving average
AVG(amount) OVER (ORDER BY order_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)

-- Share of total
100.0 * amount / SUM(amount) OVER ()                -- empty OVER() = whole result set
```

**Frame trap:** with `ORDER BY` and no explicit frame the default is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, which treats rows with **equal ORDER BY values as peers** and gives them the same running total. Write `ROWS BETWEEN ...` explicitly for a true row-by-row running sum. `FIRST_VALUE`, `LAST_VALUE` (default frame stops at the current row, so `LAST_VALUE` needs `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`).

### Duplicates: find and delete

```sql
-- Find duplicate emails
SELECT email, COUNT(*) FROM users GROUP BY email HAVING COUNT(*) > 1;

-- Show the duplicate rows themselves
SELECT * FROM (SELECT *, COUNT(*) OVER (PARTITION BY email) AS c FROM users) t WHERE c > 1;

-- Delete duplicates, keep the lowest id (works in PostgreSQL, SQL Server, SQLite)
DELETE FROM users WHERE id IN (
  SELECT id FROM (SELECT id, ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) AS rn FROM users) t WHERE rn > 1
);
-- MySQL: cannot select from the table being deleted unless wrapped in a derived table (as above) or use a self-join:
DELETE u1 FROM users u1 JOIN users u2 ON u1.email = u2.email AND u1.id > u2.id;
```

### Consecutive dates, gaps and islands

**Island trick:** within a run of consecutive dates, `date - ROW_NUMBER()` is constant, so it labels each run.

```sql
-- logins(user_id, login_date): user 1 logged in Jan 1, 2, 3, 5, 6, 9
WITH d AS (
  SELECT DISTINCT user_id, login_date FROM logins                   -- dedupe first, or ROW_NUMBER breaks
), g AS (
  SELECT user_id, login_date,
         login_date - CAST(ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) AS INT) AS grp   -- PostgreSQL date arithmetic
         -- MySQL: DATE_SUB(login_date, INTERVAL ROW_NUMBER() OVER (...) DAY)
  FROM d
)
SELECT user_id, MIN(login_date) AS streak_start, MAX(login_date) AS streak_end, COUNT(*) AS streak_len
FROM g GROUP BY user_id, grp ORDER BY user_id, streak_start;
-- user 1: Jan1..Jan3 (3), Jan5..Jan6 (2), Jan9..Jan9 (1)
-- Longest streak per user: add MAX(streak_len) over this result.

-- Gaps: where a run of dates is broken
SELECT user_id, login_date AS last_day_before_gap, next_date
FROM (SELECT user_id, login_date, LEAD(login_date) OVER (PARTITION BY user_id ORDER BY login_date) AS next_date FROM d) t
WHERE next_date - login_date > 1;

-- Numbers appearing at least 3 times consecutively
SELECT DISTINCT num FROM (
  SELECT num, LAG(num) OVER (ORDER BY id) p1, LAG(num, 2) OVER (ORDER BY id) p2 FROM logs
) t WHERE num = p1 AND num = p2;
```

### Retention queries

```sql
-- Day-1 retention: share of users who came back the day after their first activity
WITH first_day AS (SELECT user_id, MIN(activity_date) AS d0 FROM activity GROUP BY user_id)
SELECT ROUND(1.0 * COUNT(DISTINCT a.user_id) / COUNT(DISTINCT f.user_id), 2) AS d1_retention
FROM first_day f LEFT JOIN activity a ON a.user_id = f.user_id AND a.activity_date = f.d0 + 1;   -- MySQL: DATE_ADD(f.d0, INTERVAL 1 DAY)

-- Monthly cohort retention (PostgreSQL)
WITH c AS (SELECT user_id, DATE_TRUNC('month', MIN(activity_date)) AS cohort FROM activity GROUP BY user_id)
SELECT c.cohort,
       (EXTRACT(YEAR FROM age(DATE_TRUNC('month', a.activity_date), c.cohort)) * 12 +
        EXTRACT(MONTH FROM age(DATE_TRUNC('month', a.activity_date), c.cohort)))::int AS month_no,
       COUNT(DISTINCT a.user_id) AS active_users
FROM c JOIN activity a USING (user_id) GROUP BY 1, 2 ORDER BY 1, 2;    -- divide by month_no = 0 count for the %
```

### Pagination: OFFSET vs keyset

```sql
-- OFFSET pagination (page 1000, 20 per page)
SELECT * FROM posts ORDER BY created_at DESC, id DESC LIMIT 20 OFFSET 19980;

-- Keyset (seek) pagination: continue after the last row of the previous page
SELECT * FROM posts
WHERE (created_at, id) < (:last_created_at, :last_id)        -- row-value comparison; expand manually if unsupported
ORDER BY created_at DESC, id DESC LIMIT 20;                  -- backed by an index on (created_at, id)
```

|  | OFFSET | Keyset |
| --- | --- | --- |
| Cost for page N | O(offset + limit): the database reads and discards all skipped rows | O(limit) with an index |
| Consistency | rows shift if data is inserted or deleted between requests (duplicates or skips) | stable |
| Jump to arbitrary page | yes | no (next/previous only) |
| Requirement | none | a unique, indexed, deterministic sort key (always add `id` as a tie-breaker) |

**More classics to practise:** median salary (`PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary)`, or `ROW_NUMBER` from both ends), pivot rows to columns with conditional aggregation, customers who never ordered (anti-join), overlapping intervals (`a.start < b.end AND b.start < a.end`), departments where all employees earn above X (`HAVING MIN(salary) > X`), month-over-month growth (`LAG`), first and last order per customer (`FIRST_VALUE`/`ROW_NUMBER`), cumulative distribution, swap two column values with `CASE`.

---

## S5. Database design and performance in SQL: views, procedures, triggers, indexes, EXPLAIN, transactions

Theory of keys, normalization, B+ trees, isolation and optimization is in Part 2 (B3 to B10); this section is the SQL syntax and the interview answers.

**Views**

```sql
CREATE VIEW dept_summary AS
SELECT d.dept_name, COUNT(e.emp_id) AS headcount, AVG(e.salary) AS avg_salary
FROM departments d LEFT JOIN employees e ON e.dept_id = d.dept_id GROUP BY d.dept_name;
SELECT * FROM dept_summary WHERE headcount > 0;
```

A view is a **stored query**, a virtual table (no data of its own) used for abstraction, simplification and **security** (expose only some columns/rows). **Materialized views** store the result physically and need refreshing (`REFRESH MATERIALIZED VIEW`), trading freshness for speed. Simple views are updatable; views with joins, aggregates or `DISTINCT` usually are not. A view does not make a slow query faster by itself.

**Stored procedures vs functions vs triggers**

|  | Stored procedure | Function | Trigger |
| --- | --- | --- | --- |
| Invoked by | `CALL proc(...)` | inside a query: `SELECT fn(x)` | automatically on `INSERT/UPDATE/DELETE` |
| Returns | nothing, or OUT parameters / result sets | a value (scalar or table) | nothing |
| Transactions | can control them (`COMMIT` inside, DB dependent) | usually not | runs inside the triggering statement's transaction |
| Use | multi-step business logic close to the data | reusable computations | auditing, derived data, enforcing rules |

```sql
-- MySQL procedure
DELIMITER //
CREATE PROCEDURE give_raise(IN p_dept INT, IN p_pct DECIMAL(5,2))
BEGIN
  UPDATE employees SET salary = salary * (1 + p_pct / 100) WHERE dept_id = p_dept;
END //
DELIMITER ;
CALL give_raise(2, 10);

-- PostgreSQL audit trigger (sketch)
CREATE FUNCTION log_salary() RETURNS trigger AS $$
BEGIN
  INSERT INTO salary_audit(emp_id, old_salary, new_salary, changed_at) VALUES (OLD.emp_id, OLD.salary, NEW.salary, now());
  RETURN NEW;
END $$ LANGUAGE plpgsql;
CREATE TRIGGER trg_salary AFTER UPDATE OF salary ON employees FOR EACH ROW EXECUTE FUNCTION log_salary();
```

**Pros of procedures:** fewer network round trips, central logic, precompiled plans, permission control. **Cons:** business logic hidden in the database, harder to version-control/test, vendor lock-in. **Trigger pitfalls:** invisible side effects, cascading triggers, hurt bulk performance, hard to debug.

**Indexes in SQL**

```sql
CREATE INDEX idx_emp_dept ON employees(dept_id);
CREATE UNIQUE INDEX uq_email ON users(email);
CREATE INDEX idx_orders_cust_date ON orders(customer_id, order_date);       -- composite: leftmost-prefix rule
CREATE INDEX idx_paid ON orders(order_date) WHERE status = 'PAID';          -- partial index (PostgreSQL)
DROP INDEX idx_emp_dept;                                                    -- MySQL: DROP INDEX idx ON table
```

Clustered vs non-clustered, covering indexes and when the optimizer ignores an index are covered in Part 2, B6. Quick rules: index foreign keys and columns used in joins and filters; put equality columns first in a composite index and the range column last; verify with `EXPLAIN`.

**EXPLAIN**

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 101 AND order_date >= '2026-01-02';
EXPLAIN ANALYZE ...        -- PostgreSQL/MySQL 8: actually executes and reports real times and row counts
```

Read: **access type** (full scan `Seq Scan`/`ALL` vs `Index Scan`/`ref`/`range`), **chosen index**, **estimated vs actual rows** (large gaps mean stale statistics: run `ANALYZE`), **join order and algorithm** (nested loop, hash join, merge join), **sort or temp table** steps (`Using filesort`, `Using temporary` in MySQL) that an index could remove. Fix order: make predicates sargable, add or adjust an index, rewrite the query, refresh statistics.

**Transactions in SQL**

```sql
BEGIN;                                        -- or START TRANSACTION
UPDATE accounts SET bal = bal - 100 WHERE id = 1;
SAVEPOINT after_debit;
UPDATE accounts SET bal = bal + 100 WHERE id = 2;
-- if something fails: ROLLBACK TO SAVEPOINT after_debit;  (or ROLLBACK to undo everything)
COMMIT;
```

Isolation levels, anomalies, MVCC and `SELECT ... FOR UPDATE` are in Part 2, B7. Remember that DDL implicitly commits in MySQL.

**SQL injection (security essential)**

```python
# VULNERABLE: input becomes part of the SQL text. name = "x' OR '1'='1" returns every row.
cur.execute(f"SELECT * FROM users WHERE name = '{name}'")

# SAFE: parameterised query: the driver sends the value separately from the SQL text
cur.execute('SELECT * FROM users WHERE name = %s', (name,))
```

Always use **prepared/parameterised statements**; never build SQL by concatenation. Add least-privilege database accounts, input validation and an allow-list for dynamic column or table names (identifiers cannot be parameterised).

**Interview traps:** a view is not stored data (unless materialized); `TRUNCATE` is DDL and usually cannot be rolled back in MySQL; triggers fire per row and slow bulk loads; `SELECT COUNT(*)` on a big table is slow even with an index in some engines; adding an index can slow writes; wrapping a column in a function disables its index.

---

## S6. SQL practice set with answers (run on the S0 data)

1. **Employees with no department.** `SELECT name FROM employees WHERE dept_id IS NULL;` → Omar.
2. **Departments with no employees.** `SELECT dept_name FROM departments d WHERE NOT EXISTS (SELECT 1 FROM employees e WHERE e.dept_id = d.dept_id);` → Legal.
3. **Second highest salary.** DENSE\_RANK version in S4 → 90000.
4. **Department with the highest average salary.** `SELECT dept_id FROM employees WHERE dept_id IS NOT NULL GROUP BY dept_id ORDER BY AVG(salary) DESC LIMIT 1;` → dept 3 (HR, 95000); to include ties use `HAVING AVG(salary) = (SELECT MAX(a) FROM (SELECT AVG(salary) a FROM employees GROUP BY dept_id) t)`.
5. **Managers with at least two direct reports.** `SELECT m.name FROM employees m JOIN employees e ON e.manager_id = m.emp_id GROUP BY m.emp_id, m.name HAVING COUNT(*) >= 2;` → Asha (4 reports), Kiran (2).
6. **Employees earning more than their department average.** Correlated subquery in S3, or `SELECT name FROM (SELECT name, salary, AVG(salary) OVER (PARTITION BY dept_id) a FROM employees) t WHERE salary > a;` → Asha and Sana (Neha is alone in HR and Omar has no department, so each equals their own group's average and is excluded).
7. **Top 2 earners per department.** S4 (use `ROW_NUMBER` for exactly 2 rows, `DENSE_RANK` to keep ties).
8. **Headcount for every department including empty ones.** LEFT JOIN + `COUNT(e.emp_id)` in S2.
9. **Total paid amount per customer, and customers with total above 500.** `SELECT customer_id, SUM(amount) t FROM orders WHERE status = 'PAID' GROUP BY customer_id HAVING SUM(amount) > 500;` → 101 (900), 102 (900).
10. **Running total of paid amount ordered by date.** `SELECT order_date, amount, SUM(amount) OVER (ORDER BY order_date, order_id ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) FROM orders WHERE status = 'PAID';` (the extra `order_id` makes ties deterministic).
11. **Day-over-day change in daily revenue.** `WITH daily AS (SELECT order_date, SUM(amount) r FROM orders WHERE status='PAID' GROUP BY order_date) SELECT order_date, r, r - LAG(r) OVER (ORDER BY order_date) AS delta FROM daily;`
12. **Customers who ordered on at least two consecutive days.** `SELECT DISTINCT a.customer_id FROM orders a JOIN orders b ON a.customer_id = b.customer_id AND b.order_date = a.order_date + 1;` (MySQL: `DATE_ADD(a.order_date, INTERVAL 1 DAY)`) → 101.
13. **Pivot status counts per customer.** Conditional aggregation in S3.
14. **Delete duplicate rows keeping the earliest.** S4 `ROW_NUMBER` delete.
15. **Median salary.** PostgreSQL: `SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) FROM employees;` portable: number rows ascending and descending with `ROW_NUMBER`, keep rows where the two numbers differ by at most 1, then `AVG`.

**Habits that win SQL rounds:** state assumptions about NULLs and ties out loud; build the query in steps (CTEs); test on a tiny example mentally; mention the index that would make it fast; know the dialect (LIMIT vs TOP, `DATE_ADD` vs `+ interval`, `FULL JOIN` support).

---

# PART G: GIT AND COLLABORATIVE DEVELOPMENT

## G1. Mental model: how Git actually works

**Git is a distributed version control system.** Every clone holds the **full history**, so most operations are local and fast.

```
 Working directory  --git add-->  Staging area (index)  --git commit-->  Local repository (.git)  --git push-->  Remote (origin)
   (your files)     <--restore--   (next snapshot)      <--reset--        (full history)          <--fetch/pull--  (GitHub, GitLab)
```

**Objects (stored by SHA hash, content-addressable):** **blob** (file contents), **tree** (a directory: names pointing to blobs/trees), **commit** (pointer to a tree + parent commit(s) + author + message), **tag** (annotated name for a commit). A commit is a **snapshot** of the project, not a diff (diffs are computed on demand). Identical content shares one blob.

**References:** a **branch** is just a movable pointer (a 41-byte file) to a commit. **HEAD** points to the current branch (or directly to a commit in *detached HEAD* state). Committing moves the current branch pointer forward.

```
A <- B <- C <- D        main, HEAD -> main
           \
            E <- F      feature   (branch = a label on a commit; creating one is instant and cheap)
```

**Interview answer:** "Git stores snapshots as immutable, hash-identified objects; branches and HEAD are lightweight pointers, which is why branching and merging are cheap. The staging area lets me build exactly the next commit."

**Loopholes:** Git tracks content, not empty directories (add a `.gitkeep`); a commit hash identifies content **and** history, so changing anything (amend, rebase) creates new hashes; Git is not GitHub (GitHub is a hosting service with pull requests and CI).

---

## G2. Everyday commands

```bash
git init                              # create a repository in the current folder
git clone https://github.com/u/repo.git   # copy a remote repo (history + 'origin' remote + default branch checked out)
git status                            # what is modified, staged, untracked
git add file.py        git add -p     # stage a file; -p stages chosen hunks interactively
git add .                             # stage everything (check .gitignore first)
git commit -m 'Fix null check in parser'   # record the staged snapshot
git commit --amend                    # rewrite the LAST commit (message or contents); new hash
git log --oneline --graph --decorate --all     # compact history with branch shapes
git log -p -3        git log --stat   git log --author=asha --since='2 weeks ago'
git diff                              # working tree vs index (unstaged changes)
git diff --staged                     # index vs last commit (what will be committed)
git diff main..feature                # difference between two branches/commits
git branch                            # list local branches;  git branch -a  includes remote ones
git branch feature-x                  # create;  git branch -d feature-x  delete (merged);  -D force
git switch feature-x                  # change branch;  git switch -c new-branch  create and switch
git checkout feature-x                # older command that does many things (branches AND files); prefer switch/restore
git merge feature-x                   # merge feature-x INTO the current branch
git fetch origin                      # download remote commits, do NOT change your branches
git pull                              # = fetch + merge (or git pull --rebase = fetch + rebase)
git push -u origin feature-x          # publish a branch and set upstream tracking
git remote -v                         # list remotes;  git remote add origin <url>
```

**Fetch vs pull:** `fetch` is safe (updates `origin/main` only); `pull` also integrates into your current branch and may create a merge commit or conflicts. Fetch first when you want to look before merging.

**Typical session**

```bash
git switch -c feature/login          # branch off
# ...edit...
git add -p && git commit -m 'Add login form validation'
git fetch origin && git rebase origin/main     # sync with the latest main
git push -u origin feature/login     # open a pull request on GitHub
```

**Commit message conventions:** short imperative summary (about 50 characters), blank line, body explaining **why**; or Conventional Commits: `feat: add OTP login`, `fix: handle empty cart`, `docs:`, `refactor:`, `test:`, `chore:`. Make commits **atomic** (one logical change, builds and passes tests).

**Loopholes:** `git add .` can stage secrets and build output; `git checkout <file>` overwrites local changes silently; `git pull` on a dirty tree can surprise you; commit identity comes from `git config user.name` / `user.email` (set them, per repo or globally).

---

## G3. The undo toolbox: restore, reset, revert, reflog

**Three-area view of `git reset <commit>`**

| Mode | Moves HEAD/branch | Resets staging area | Resets working directory | Typical use |
| --- | --- | --- | --- | --- |
| `--soft` | yes | no (changes stay **staged**) | no | redo the last commit(s): squash or reword |
| `--mixed` (default) | yes | **yes** (changes become **unstaged**) | no | unstage everything and re-pick |
| `--hard` | yes | yes | **yes** (changes are **discarded**) | throw work away completely (dangerous) |

```bash
git reset --soft HEAD~1              # undo last commit, keep changes staged
git reset HEAD~1                     # undo last commit, keep changes in files (unstaged)
git reset --hard HEAD~1              # undo last commit AND delete its changes
git reset --hard origin/main         # make local branch identical to the remote (discards local work)
git restore file.py                  # discard UNSTAGED edits of a file (restore from index)
git restore --staged file.py         # unstage a file (keep edits)
git restore --source=HEAD~2 file.py  # bring back an old version of a file
git revert <commit>                  # NEW commit that applies the inverse of <commit>
git revert -m 1 <merge-commit>       # revert a merge (choose mainline parent)
git clean -fd                        # delete untracked files/dirs (use -n first for a dry run)
```

**reset vs revert (a top interview question)**

|  | `reset` | `revert` |
| --- | --- | --- |
| What it does | moves the branch pointer back, **rewriting history** | adds a **new commit** that undoes an old one; history stays intact |
| Safe on shared/pushed branches | **no** (needs force push, breaks collaborators) | **yes** |
| Use for | local, unpublished mistakes | undoing something already pushed to main |

**Reflog: the safety net.** `git reflog` lists where `HEAD` (and each branch) pointed over time, including commits that no branch references any more.

```bash
git reflog                           # e.g. a1b2c3d HEAD@{2}: commit: add payment retry
git reset --hard HEAD@{2}            # go back to that state
git branch rescue a1b2c3d            # or just attach a branch to the lost commit
# Recover a deleted branch:  git reflog  ->  find its last commit  ->  git switch -c branch-name <sha>
```

Reflog entries live about 90 days (reachable) / 30 days (unreachable) by default and are **local only**. Committed work is almost always recoverable; **uncommitted** changes destroyed by `reset --hard` or `restore` are not.

**Decision table**

| Situation | Command |
| --- | --- |
| Typo in the last commit message | `git commit --amend` |
| Forgot a file in the last commit | `git add file && git commit --amend --no-edit` |
| Unstage a file | `git restore --staged file` |
| Discard local edits to a file | `git restore file` |
| Undo last local commit, keep the work | `git reset --soft HEAD~1` |
| Undo a bad commit already on `main` | `git revert <sha>` |
| Lost a commit after reset/rebase | `git reflog` then `git reset --hard <sha>` or branch it |

**Loopholes:** `reset --hard` is the main way people lose uncommitted work; amending or resetting **pushed** commits forces `git push --force`, which overwrites teammates' work (prefer `--force-with-lease`, which refuses if the remote moved); `git revert` of a merge needs `-m`, and re-merging the branch later requires reverting the revert.

---

## G4. Branching, merge vs rebase, and conflicts

**Merge**

```
Fast-forward (main has not moved): main just slides forward
  A--B--C                  A--B--C--D--E
         \            ->                ^ main, feature
          D--E feature

Three-way merge (both moved): creates a MERGE COMMIT with two parents
  A--B--C--F  main                A--B--C--F--M   main
      \                     ->        \        /
       D--E   feature                  D------E
```

`git merge --no-ff feature` forces a merge commit even when a fast-forward is possible (keeps the branch visible in history). `git merge --squash feature` stages all changes as one commit without a merge link.

**Rebase: replay your commits on top of another base**

```
Before:  A--B--C  main          After `git switch feature && git rebase main`:
             \                  A--B--C  main
              D--E  feature              \
                                          D'--E'  feature   (new commits, new hashes)
```

|  | Merge | Rebase |
| --- | --- | --- |
| History | preserves true history, adds merge commits (non-linear) | **linear** history, rewrites commits |
| Safety | non-destructive | rewrites history: **never rebase commits others have pulled** |
| Conflicts | resolve once | may resolve repeatedly, per replayed commit |
| Best for | integrating a finished feature into `main`; shared branches | cleaning up **your own local** branch before a PR; staying up to date with main |

**Golden rule:** do not rebase public/shared history. Rebase your private feature branch; merge into shared branches.

**Interactive rebase (clean up before the PR)**

```bash
git rebase -i HEAD~4        # opens a todo list of the last 4 commits
# pick    keep the commit          reword  change the message          edit  stop to amend
# squash  meld into previous (keep both messages)   fixup  meld, discard its message      drop  delete the commit
# reorder lines to reorder commits.   git rebase --abort  cancels;   git rebase --continue  after fixing conflicts
```

**Resolving merge conflicts**

A conflict occurs when both sides changed the same lines (or one side deleted a file the other edited). Git marks the file:

```
<<<<<<< HEAD
return price * 0.9;          # your side (current branch)
=======
return price * 0.85;         # incoming side
>>>>>>> feature-x
```

```bash
git status                    # lists 'both modified' files
# edit: keep one side, the other, or combine; delete ALL marker lines; build and run tests
git add path/to/file          # mark resolved
git commit                    # finishes a merge  (for rebase: git rebase --continue)
git merge --abort             # give up and return to the pre-merge state
git checkout --ours file      # take your version   (--theirs: incoming)   # note: in a REBASE the meanings swap
git config rerere.enabled true    # remember and reuse resolutions
git mergetool                 # visual three-way tool
```

**Reducing conflicts:** small, short-lived branches; pull/rebase often; agree on formatters (auto-format on commit) so whitespace is not a conflict; avoid mass reformatting and file moves in feature branches; communicate about shared files; split large files.

**Loopholes:** conflict markers committed by accident (run a search for `<<<<<<<` and have CI check); resolving by blindly taking one side loses the other's work; "ours/theirs" are reversed during a rebase; `git pull` creating noisy merge commits (use `pull --rebase` or `git config pull.rebase true`); a clean merge can still be semantically broken, so always run tests after merging.

---

## G5. Stash, cherry-pick, bisect and tags

**Stash:** shelve uncommitted work temporarily.

```bash
git stash push -m 'wip: parser'     # save tracked changes (add -u to include untracked files)
git stash list                      # stash@{0}: On main: wip: parser
git stash pop                       # reapply the newest stash AND drop it
git stash apply stash@{1}           # reapply but keep it in the list
git stash drop stash@{0}   git stash clear
git stash branch fix-x stash@{0}    # create a branch from a stash (useful if it conflicts)
```

Use it to switch branches quickly for a hot fix. Stashes are local, easy to forget, and `pop` can conflict (the stash then stays so you do not lose it).

**Cherry-pick:** copy specific commits onto the current branch (new hashes).

```bash
git switch release/1.2
git cherry-pick 9fceb02             # apply one commit;  A^..B for a range;  -x appends the original hash to the message
git cherry-pick --continue / --abort    # after / during conflicts
```

Uses: backport a bug fix to a release branch; rescue a commit made on the wrong branch (`cherry-pick` then `reset` the wrong branch). Downside: duplicates commits, so a later merge may conflict or duplicate changes.

**Bisect:** binary search through history to find the commit that introduced a bug in O(log n) steps.

```bash
git bisect start
git bisect bad                      # current commit is broken
git bisect good v1.4.0              # this old one worked
# Git checks out the middle commit; test it, then run:  git bisect good   or   git bisect bad
git bisect run ./test.sh            # fully automatic: script exits 0 = good, 1-127 = bad
git bisect reset                    # return to the original branch
```

Requires the bug to be reproducible and commits to be buildable (atomic commits matter). 1000 commits need about 10 steps.

**Tags and release versioning**

```bash
git tag v1.2.0                      # lightweight tag (just a pointer)
git tag -a v1.2.0 -m 'Release 1.2.0'    # annotated tag (tagger, date, message; recommended for releases; can be signed with -s)
git push origin v1.2.0   git push --tags    # tags are NOT pushed by default
git tag -d v1.2.0   git push origin --delete v1.2.0
git describe --tags                 # nearest tag + commits since
```

**Semantic versioning `MAJOR.MINOR.PATCH`:** MAJOR = breaking change, MINOR = backward-compatible feature, PATCH = backward-compatible fix (`2.4.1` → `2.4.2` for a fix, `2.5.0` for a feature, `3.0.0` for a break); pre-releases `1.0.0-rc.1`. CI can build and publish artifacts when a `v*` tag is pushed.

**Other handy commands:** `git blame file` (who last changed each line), `git show <sha>`, `git log -S 'text'` (find when a string was added), `git shortlog -sn`, `git worktree add ../hotfix main` (second working copy of the same repo), `git config --global alias.st status`.

---

## G6. Team workflow: `.gitignore`, pull requests, reviews, branching strategies

**`.gitignore` and repository hygiene**

```
# .gitignore
node_modules/
__pycache__/
*.pyc
.env                 # secrets and local config: NEVER commit
*.log
build/
dist/
.idea/
.vscode/
!important.log       # '!' re-includes a file
```

- Patterns: `*.log`, `dir/` (directories only), `/rootonly` (anchored), `**/temp`, `!` to negate.
- **A file already tracked is not ignored.** Untrack it: `git rm --cached file` (or `-r dir`), then commit.
- Use a **global ignore** for editor/OS junk (`git config --global core.excludesfile ~/.gitignore_global`).
- **Commit `.env.example`, not `.env`.** Never commit credentials, API keys, private keys, large binaries, build output, dependency folders.
- **If a secret is committed:** **rotate/revoke it immediately** (it is compromised the moment it is pushed, even if you delete it later), then remove it from history with `git filter-repo` or BFG and force-push, and tell collaborators to re-clone. Enable secret scanning.
- Hygiene: a useful `README` (what, why, how to run, tests, screenshots), `LICENSE`, `CONTRIBUTING.md`, consistent formatting/linting (pre-commit hooks), small focused commits, no commented-out code dumps, a clean default branch that always builds.

**Pull request (PR) workflow**

1. Branch from up-to-date `main`; make small, focused commits.
2. Push and open a PR with a clear title, **what/why**, testing notes, screenshots, linked issue (`Fixes #123`).
3. CI runs (build, lint, tests); reviewers comment; you push fixes to the same branch.
4. Approvals + green checks + no conflicts → merge.
5. Delete the branch; the issue closes automatically.

**Merge methods on GitHub:** *merge commit* (keeps all commits + a merge commit), *squash and merge* (one clean commit on main; hides the branch's internal history), *rebase and merge* (linear, individual commits replayed). Squash is popular for tidy history; it loses individual commit granularity.

**Branch protection:** require PR reviews, required status checks, up-to-date branch, signed commits, no force-push or deletion of `main`, `CODEOWNERS` to auto-request the right reviewers.

**Code review practice**

- *As an author:* keep PRs small (a few hundred lines), self-review first, explain context, add tests, respond to every comment, do not take it personally.
- *As a reviewer:* check correctness, edge cases, readability, tests, security (injection, secrets, authz), performance and complexity, API and naming; separate **blocking** issues from **nits**; be specific, kind and ask questions rather than dictate; approve quickly when it is good enough.

**Branching strategies**

| Strategy | How | Pros | Cons / best for |
| --- | --- | --- | --- |
| **Git Flow** | long-lived `main` + `develop`; `feature/*`, `release/*`, `hotfix/*` | structured releases, parallel versions | heavy, slow integration; scheduled-release products |
| **GitHub Flow** | `main` is always deployable; short-lived feature branches → PR → merge → deploy | simple, fast, great with CI/CD | needs strong tests; web apps/SaaS |
| **Trunk-based development** | everyone commits to `main` (or very short branches, under a day or two); unfinished work hidden behind **feature flags** | minimal merge pain, continuous integration | needs discipline, flags and automated tests; high-velocity teams |
| **Release branches** (+ cherry-pick fixes) | cut `release/x.y` to stabilize | supports old versions | backporting overhead |

**Loopholes:** ignoring a file after it was committed does nothing until you `git rm --cached` it; long-lived feature branches create painful merges; huge PRs get rubber-stamped; squash-merging loses authorship granularity but not authorship itself; do not force-push shared branches.

---

## G7. GitHub Actions and CI/CD basics

**CI (continuous integration):** every push or PR automatically builds and tests the code so integration problems appear within minutes. **CD:** *continuous delivery* = every passing change is packaged and **ready** to deploy (a human approves the release); *continuous deployment* = every passing change is **deployed automatically**.

```
commit -> push/PR -> [build -> lint -> unit tests -> integration tests -> security scan] -> artifact/image -> deploy staging -> (approval) -> deploy production -> monitor
```

**GitHub Actions vocabulary:** a **workflow** is a YAML file in `.github/workflows/`; it starts on an **event** (`push`, `pull_request`, `schedule`, `workflow_dispatch`, `release`); it contains **jobs** (run in parallel unless `needs:` orders them), each running on a **runner** (`ubuntu-latest`, or self-hosted) as a sequence of **steps**, which run shell commands (`run:`) or reusable **actions** (`uses:`). Also: **secrets**, **matrix** builds, **cache**, **artifacts**, **environments** with required approvals.

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read                      # least privilege for the GITHUB_TOKEN

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.10', '3.11', '3.12']     # runs the job once per version, in parallel
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: pip                  # caches dependencies between runs
      - run: pip install -r requirements.txt
      - run: pytest --maxfail=1 -q

  build-and-deploy:
    needs: test                       # only after tests pass
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production           # can require manual approval and scoped secrets
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t myapp:${{ github.sha }} .
      - run: echo '${{ secrets.REGISTRY_TOKEN }}' | docker login ghcr.io -u ${{ github.actor }} --password-stdin
      - run: docker push ghcr.io/${{ github.repository }}:${{ github.sha }}
```

A C++ job is the same with `sudo apt-get install -y cmake`, `cmake -B build && cmake --build build`, `ctest --test-dir build`.

**Best practices:** pin third-party actions to a version or commit SHA; set minimal `permissions`; store credentials in **secrets** (never in YAML) and never echo them; cache dependencies; keep the pipeline fast (parallel jobs, fail fast); require the CI check to pass in branch protection; build the artifact **once** and promote the same artifact through environments; add rollback and health checks; use environments with approvals for production; publish test and coverage reports as artifacts.

**Interview answer:** "On every PR the workflow checks out the code, installs dependencies with caching, runs lint and tests in a matrix, and blocks merge on failure. On merges to main it builds a Docker image tagged with the commit SHA, pushes it to a registry and deploys to staging, then production after approval, with secrets in the environment store."

**Loopholes:** secrets are not available to workflows triggered from forks' pull requests (by design); `pull_request_target` runs with secrets and is a known security risk if it checks out untrusted code; a green pipeline proves only what the tests cover; flaky tests erode trust, so quarantine and fix them; using `latest` image tags makes deployments non-reproducible.

---

## G8. Scenario Q&A and command cheat sheet

1. **I committed but want to undo it and keep my changes.** `git reset --soft HEAD~1` (staged) or `git reset HEAD~1` (unstaged).
2. **I committed to `main` by mistake instead of a feature branch (not pushed).** `git switch -c feature` (branch keeps the commit), then `git switch main && git reset --hard origin/main`.
3. **I pushed a bad commit to a shared branch.** `git revert <sha>` and push; do not rewrite shared history.
4. **I pushed a password.** Revoke and rotate it now; then purge with `git filter-repo`/BFG, force-push, notify the team. Deleting the file in a new commit does not remove it from history.
5. **I lost commits after a hard reset or a bad rebase.** `git reflog`, find the SHA, `git reset --hard <sha>` (or `git branch rescue <sha>`).
6. **I deleted a branch accidentally.** `git reflog` to find its tip, `git switch -c name <sha>`.
7. **What is a detached HEAD?** HEAD points at a commit rather than a branch (after `git checkout <sha>` or a tag). New commits belong to no branch and can be lost; save with `git switch -c new-branch`.
8. **`git push` says non-fast-forward / rejected.** The remote has commits you do not have. `git pull --rebase` (or fetch + merge), resolve, push. Use `--force-with-lease` only on your own branch.
9. **Combine the last 3 commits into one.** `git rebase -i HEAD~3` and `squash`/`fixup`, or `git reset --soft HEAD~3 && git commit`.
10. **Merge or rebase?** Rebase your private branch to update and tidy; merge to integrate finished work into shared branches; never rebase shared history.
11. **How do I find which commit broke the build?** `git bisect start`, mark bad and good, or `git bisect run <test>`.
12. **`.gitignore` is not working.** The file is already tracked: `git rm --cached file`, commit; check the pattern with `git check-ignore -v file`.
13. **Undo a merge that is already pushed.** `git revert -m 1 <merge-sha>`.
14. **See what changed in a teammate's PR locally.** `git fetch origin pull/123/head:pr-123 && git switch pr-123` (GitHub) or `git fetch && git diff main...origin/feature`.
15. **Rename the current branch / change remote URL.** `git branch -m new-name`; `git remote set-url origin <url>`.
16. **My stash pop conflicts.** Resolve like a merge conflict; the stash is kept until you drop it.
17. **Two or more people edited the same file.** Whoever merges second resolves conflicts: pull latest, resolve, run tests, push.
18. **Commit only part of a file.** `git add -p`.

**Cheat sheet**

| Goal | Command |
| --- | --- |
| Start | `git init`, `git clone <url>` |
| Save work | `git add -p`, `git commit -m '...'` |
| Inspect | `git status`, `git log --oneline --graph`, `git diff [--staged]`, `git show`, `git blame` |
| Branch | `git switch -c b`, `git branch -d b`, `git merge b`, `git rebase main` |
| Sync | `git fetch`, `git pull --rebase`, `git push -u origin b` |
| Undo | `git restore`, `git restore --staged`, `git reset --soft/--mixed/--hard`, `git revert`, `git commit --amend` |
| Recover | `git reflog` |
| Shelve | `git stash push/pop/list` |
| Copy a commit | `git cherry-pick <sha>` |
| Find a bug | `git bisect start/good/bad/run` |
| Release | `git tag -a v1.0.0 -m '...'`, `git push --tags` |
| Clean | `git clean -nfd` (dry run first), `git gc` |

---

*End of Part 3. Part 4 covers OOP, SOLID, design patterns, UML and low-level design problems, with both C++ and Python code.*
