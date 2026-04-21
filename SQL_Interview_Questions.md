# Top 30 SQL Interview Questions — Senior Developer

---

## Category Index

1. [Joins & Subqueries](#joins--subqueries) — Q1–Q6
2. [Window Functions](#window-functions) — Q7–Q12
3. [Performance & Indexing](#performance--indexing) — Q13–Q18
4. [Transactions & Locking](#transactions--locking) — Q19–Q22
5. [Schema Design & Normalization](#schema-design--normalization) — Q23–Q26
6. [Advanced & Tricky](#advanced--tricky) — Q27–Q30

---

## Joins & Subqueries

---

### Q1. What is the difference between INNER JOIN, LEFT JOIN, RIGHT JOIN, and FULL OUTER JOIN?

**Answer:**

| Join Type | Returns |
|---|---|
| `INNER JOIN` | Only rows that match in **both** tables |
| `LEFT JOIN` | All rows from left table + matching rows from right (NULL if no match) |
| `RIGHT JOIN` | All rows from right table + matching rows from left (NULL if no match) |
| `FULL OUTER JOIN` | All rows from both tables (NULL on either side when no match) |

```sql
-- Employees who have a department (INNER)
SELECT e.name, d.name FROM employees e
INNER JOIN departments d ON e.dept_id = d.id;

-- All employees, even those without a department (LEFT)
SELECT e.name, d.name FROM employees e
LEFT JOIN departments d ON e.dept_id = d.id;

-- Find employees with NO department
SELECT e.name FROM employees e
LEFT JOIN departments d ON e.dept_id = d.id
WHERE d.id IS NULL;
```

**Senior tip:** `LEFT JOIN ... WHERE right.col IS NULL` is a common pattern to find orphan records.

---

### Q2. What is a self join? Give a practical example.

**Answer:** A self join joins a table to itself. Used when rows in the same table have a parent-child or peer relationship.

```sql
-- Employee table: id, name, manager_id (manager_id references id in same table)
SELECT
    e.name   AS employee,
    m.name   AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

**Use cases:** org hierarchy, category trees, finding duplicate rows.

---

### Q3. What is the difference between EXISTS and IN? When would you prefer one over the other?

**Answer:**

| | `IN` | `EXISTS` |
|---|---|---|
| How it works | Evaluates the subquery, builds a list, checks membership | Stops as soon as **one** matching row is found |
| NULLs | `IN (NULL, ...)` — can cause unexpected false results | Not affected by NULLs |
| Performance | Better for **small** subquery result sets | Better for **large** subquery result sets |

```sql
-- IN: fine when subquery returns few rows
SELECT name FROM employees
WHERE dept_id IN (SELECT id FROM departments WHERE location = 'NYC');

-- EXISTS: better when departments table is huge
SELECT name FROM employees e
WHERE EXISTS (
    SELECT 1 FROM departments d
    WHERE d.id = e.dept_id AND d.location = 'NYC'
);
```

**Senior tip:** `NOT EXISTS` is almost always faster than `NOT IN` because `NOT IN` returns no rows if the subquery contains even one NULL.

---

### Q4. What is a correlated subquery? How does it differ from a regular subquery?

**Answer:** A correlated subquery references a column from the **outer query** — it re-executes for every row in the outer query.

```sql
-- Regular subquery: runs once
SELECT name FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);

-- Correlated subquery: runs once per row in outer query
-- "Find employees who earn more than the average salary IN THEIR OWN DEPARTMENT"
SELECT name, dept_id, salary
FROM employees e
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
    WHERE dept_id = e.dept_id   -- references outer e.dept_id
);
```

**Senior tip:** Correlated subqueries can be slow. Most can be rewritten using a `JOIN` with a derived table or window function (`AVG() OVER (PARTITION BY dept_id)`).

---

### Q5. What is the difference between WHERE and HAVING?

**Answer:**

| | `WHERE` | `HAVING` |
|---|---|---|
| When it runs | **Before** grouping — filters individual rows | **After** grouping — filters groups |
| Can use aggregates? | No | Yes |

```sql
-- WHERE filters rows before aggregation
SELECT dept_id, COUNT(*) AS headcount
FROM employees
WHERE status = 'ACTIVE'          -- filters rows first
GROUP BY dept_id
HAVING COUNT(*) > 10;            -- filters groups after aggregation
```

**Senior tip:** Put as much filtering in `WHERE` as possible — it reduces the rows before the expensive `GROUP BY` step.

---

### Q6. Write a query to find the second highest salary.

**Answer:** Multiple approaches — know all three.

```sql
-- Approach 1: OFFSET (simple, clear)
SELECT DISTINCT salary FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 1;

-- Approach 2: Subquery (works in all SQL dialects)
SELECT MAX(salary) FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);

-- Approach 3: Window function (best for Nth highest — easily parameterized)
SELECT salary FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
) ranked
WHERE rnk = 2;
```

**Senior tip:** Use `DENSE_RANK` (not `ROW_NUMBER`) so tied salaries don't skip a rank.

---

## Window Functions

---

### Q7. What are window functions? How are they different from GROUP BY?

**Answer:** Window functions compute a result **across a set of rows related to the current row** without collapsing the rows the way `GROUP BY` does.

```sql
-- GROUP BY collapses rows — you lose individual row data
SELECT dept_id, AVG(salary) FROM employees GROUP BY dept_id;

-- Window function keeps all rows + adds the department average alongside each row
SELECT
    name,
    dept_id,
    salary,
    AVG(salary) OVER (PARTITION BY dept_id) AS dept_avg,
    salary - AVG(salary) OVER (PARTITION BY dept_id) AS diff_from_avg
FROM employees;
```

**Senior tip:** `PARTITION BY` in a window function is like `GROUP BY` but without collapsing.

---

### Q8. What is the difference between RANK, DENSE_RANK, and ROW_NUMBER?

**Answer:**

Given salaries: 100, 100, 90, 80

| Function | Results | Behavior on ties |
|---|---|---|
| `ROW_NUMBER()` | 1, 2, 3, 4 | No ties — always unique |
| `RANK()` | 1, 1, 3, 4 | Ties get same rank; **next rank is skipped** |
| `DENSE_RANK()` | 1, 1, 2, 3 | Ties get same rank; **next rank is NOT skipped** |

```sql
SELECT
    name,
    salary,
    ROW_NUMBER()  OVER (ORDER BY salary DESC) AS row_num,
    RANK()        OVER (ORDER BY salary DESC) AS rnk,
    DENSE_RANK()  OVER (ORDER BY salary DESC) AS dense_rnk
FROM employees;
```

---

### Q9. What is LAG and LEAD? Give a real use case.

**Answer:** `LAG` accesses a previous row's value. `LEAD` accesses a next row's value — without a self join.

```sql
-- Month-over-month revenue change
SELECT
    month,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY month)          AS prev_month_revenue,
    revenue - LAG(revenue, 1) OVER (ORDER BY month) AS change
FROM monthly_revenue;

-- Time between consecutive orders per customer
SELECT
    customer_id,
    order_date,
    LEAD(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS next_order_date,
    DATEDIFF(
        LEAD(order_date) OVER (PARTITION BY customer_id ORDER BY order_date),
        order_date
    ) AS days_to_next_order
FROM orders;
```

---

### Q10. What is a FRAME clause in window functions? Explain ROWS vs RANGE.

**Answer:** The frame clause defines the subset of rows the window function operates on **within a partition**.

```sql
-- Running total using ROWS (physical row count)
SELECT
    order_date,
    amount,
    SUM(amount) OVER (
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total,

-- 7-day rolling average using RANGE (value-based)
    AVG(amount) OVER (
        ORDER BY order_date
        RANGE BETWEEN INTERVAL 6 DAY PRECEDING AND CURRENT ROW
    ) AS rolling_7day_avg
FROM orders;
```

| | `ROWS` | `RANGE` |
|---|---|---|
| Boundary based on | Physical row count | Value of ORDER BY column |
| Ties | Each row is a separate boundary | All tied rows are in the same boundary |

**Senior tip:** Use `ROWS` unless you specifically need value-based boundaries — `ROWS` is faster and more predictable.

---

### Q11. Write a query to find the top 1 employee per department by salary.

**Answer:**

```sql
-- Using ROW_NUMBER (most common interview answer)
SELECT dept_id, name, salary
FROM (
    SELECT
        dept_id,
        name,
        salary,
        ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn
    FROM employees
) ranked
WHERE rn = 1;

-- If you want ties included, use RANK() or DENSE_RANK() and WHERE rnk = 1
```

---

### Q12. What is NTILE? When would you use it?

**Answer:** `NTILE(n)` divides rows into `n` roughly equal buckets and assigns each row a bucket number.

```sql
-- Divide customers into 4 quartiles by total spend
SELECT
    customer_id,
    total_spend,
    NTILE(4) OVER (ORDER BY total_spend DESC) AS spend_quartile
FROM customer_summary;
-- Quartile 1 = top spenders, Quartile 4 = lowest spenders
```

**Use cases:** percentile buckets, A/B test segment assignment, customer tier classification.

---

## Performance & Indexing

---

### Q13. What is an index? What are the trade-offs of adding one?

**Answer:** An index is a separate data structure (usually a B-Tree) that lets the DB find rows without scanning the whole table.

| | Without Index | With Index |
|---|---|---|
| `SELECT` by indexed col | Full table scan — O(n) | Index seek — O(log n) |
| `INSERT / UPDATE / DELETE` | Fast — just write the row | Slower — must update index too |
| Storage | Less | More (index takes space) |

**Senior tip:** Don't index everything. Index columns used in `WHERE`, `JOIN ON`, `ORDER BY`, and foreign keys. Avoid indexing low-cardinality columns (like a boolean `is_active`).

---

### Q14. What is a composite index? What is the "leftmost prefix" rule?

**Answer:** A composite index covers multiple columns. It can be used by queries that filter on the **leftmost columns** in the index definition.

```sql
-- Index on (dept_id, status, hire_date)
CREATE INDEX idx_emp ON employees (dept_id, status, hire_date);

-- Uses index (dept_id is leftmost)
SELECT * FROM employees WHERE dept_id = 5;

-- Uses index (dept_id + status)
SELECT * FROM employees WHERE dept_id = 5 AND status = 'ACTIVE';

-- Uses index fully
SELECT * FROM employees WHERE dept_id = 5 AND status = 'ACTIVE' AND hire_date > '2020-01-01';

-- Does NOT use index (skips leftmost column)
SELECT * FROM employees WHERE status = 'ACTIVE';
```

**Senior tip:** Order columns in a composite index from most-selective (highest cardinality) to least, and match the order of your most common WHERE clauses.

---

### Q15. What is the difference between a clustered and non-clustered index?

**Answer:**

| | Clustered Index | Non-Clustered Index |
|---|---|---|
| Data storage | Table rows are **physically stored** in index order | Separate structure; contains pointers to actual rows |
| Count per table | Only **one** per table | Many allowed |
| Default in MySQL/InnoDB | Primary key is always clustered | All other indexes are non-clustered |
| Lookup | Direct — the index IS the data | Two-step: find row pointer in index, then fetch row |

**Senior tip:** In InnoDB, all non-clustered indexes store the primary key value as the row pointer. A large primary key (like a UUID) bloats every secondary index — prefer `BIGINT AUTO_INCREMENT` as PK.

---

### Q16. What does EXPLAIN do? What do you look for in the output?

**Answer:** `EXPLAIN` shows the query execution plan — how the database will execute your query.

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42 AND status = 'PENDING';
```

**Key columns to look at:**

| Column | What to look for |
|---|---|
| `type` | `const` > `ref` > `range` > `index` > `ALL` (ALL = full scan = bad) |
| `key` | Which index was used (NULL = no index used) |
| `rows` | Estimated rows examined — lower is better |
| `Extra` | `Using filesort` or `Using temporary` = performance warning |

**Senior tip:** `Using filesort` means the DB can't use the index for ordering — add the ORDER BY column to the index.

---

### Q17. What is a covering index?

**Answer:** A covering index contains **all columns** a query needs — the DB can answer the query entirely from the index without touching the actual table rows.

```sql
-- Query only needs customer_id, status, total_amount
SELECT customer_id, status, total_amount
FROM orders
WHERE customer_id = 42;

-- Covering index: includes all 3 columns
CREATE INDEX idx_orders_covering ON orders (customer_id, status, total_amount);
-- Now the query never touches the orders table rows — reads index only (much faster)
```

**Senior tip:** Look for `Using index` in EXPLAIN Extra — that means a covering index is being used.

---

### Q18. What causes a query to be slow even with an index? Name at least 4 reasons.

**Answer:**

| Reason | Example | Fix |
|---|---|---|
| **Function on indexed column** | `WHERE YEAR(created_at) = 2023` | `WHERE created_at BETWEEN '2023-01-01' AND '2023-12-31'` |
| **Implicit type cast** | `WHERE user_id = '42'` (user_id is INT) | Match data types exactly |
| **Leading wildcard** | `WHERE name LIKE '%John%'` | Use full-text index or search engine |
| **Low selectivity index** | Index on `gender` column (only 2 values) | Remove it — full scan is faster |
| **Outdated statistics** | Optimizer picks wrong plan | Run `ANALYZE TABLE` |
| **Too many rows returned** | `SELECT *` with no LIMIT on 10M rows | Add pagination / LIMIT |

---

## Transactions & Locking

---

### Q19. What are the ACID properties?

**Answer:**

| Property | Meaning | Example |
|---|---|---|
| **Atomicity** | All steps succeed or none do | Bank transfer: debit + credit both succeed or both roll back |
| **Consistency** | DB moves from one valid state to another; constraints always hold | Balance can't go negative if a CHECK constraint exists |
| **Isolation** | Concurrent transactions don't interfere with each other | Two users booking the last seat — only one succeeds |
| **Durability** | Committed data survives crashes | After `COMMIT`, data is written to disk |

---

### Q20. What are the isolation levels? What problem does each solve?

**Answer:**

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| `READ UNCOMMITTED` | Possible | Possible | Possible |
| `READ COMMITTED` | Prevented | Possible | Possible |
| `REPEATABLE READ` | Prevented | Prevented | Possible |
| `SERIALIZABLE` | Prevented | Prevented | Prevented |

- **Dirty Read:** Reading uncommitted changes from another transaction
- **Non-Repeatable Read:** Reading the same row twice gives different results (another tx updated it)
- **Phantom Read:** Re-running a range query gives different rows (another tx inserted/deleted)

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
  SELECT balance FROM accounts WHERE id = 1;  -- 1000
  -- another transaction updates balance to 800 here
  SELECT balance FROM accounts WHERE id = 1;  -- still 1000 (repeatable read)
COMMIT;
```

**Senior tip:** MySQL InnoDB default is `REPEATABLE READ`. PostgreSQL default is `READ COMMITTED`.

---

### Q21. What is a deadlock? How do you prevent it?

**Answer:** A deadlock occurs when two transactions each hold a lock the other needs — both wait forever.

```
TX1: locks Row A, waits for Row B
TX2: locks Row B, waits for Row A  ← deadlock
```

**Prevention strategies:**

| Strategy | How |
|---|---|
| **Consistent lock order** | Always lock rows/tables in the same order across all transactions |
| **Keep transactions short** | Commit quickly; don't do external calls inside a transaction |
| **Use SELECT FOR UPDATE** | Acquire locks upfront at read time instead of at write time |
| **Retry on deadlock** | Catch deadlock error (1213 in MySQL) and retry the transaction |

```sql
-- Acquiring locks upfront prevents surprise deadlocks
START TRANSACTION;
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

---

### Q22. What is the difference between optimistic and pessimistic locking?

**Answer:**

| | Pessimistic Locking | Optimistic Locking |
|---|---|---|
| Strategy | Lock the row on read; block others | No lock; detect conflict at write time |
| SQL | `SELECT ... FOR UPDATE` | Check version/timestamp on UPDATE |
| Best for | High-contention, short transactions | Low-contention, long transactions |
| Downside | Can cause deadlocks, reduces throughput | Requires retry logic on conflict |

```sql
-- Pessimistic: lock immediately
SELECT * FROM inventory WHERE product_id = 5 FOR UPDATE;
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 5;

-- Optimistic: check version at update time
-- Read: SELECT id, quantity, version FROM inventory WHERE product_id = 5
-- Write:
UPDATE inventory
SET quantity = quantity - 1, version = version + 1
WHERE product_id = 5 AND version = 3;  -- fails if someone else already updated it
-- If 0 rows affected → conflict → retry
```

---

## Schema Design & Normalization

---

### Q23. What are the normal forms? Explain 1NF, 2NF, 3NF in simple terms.

**Answer:**

| Normal Form | Rule | Violation Example |
|---|---|---|
| **1NF** | No repeating groups; each cell holds one atomic value | `phone` column stores "555-1234, 555-5678" |
| **2NF** | 1NF + every non-key column depends on the **whole** primary key (no partial dependency) | In `(order_id, product_id) → product_name`, product_name depends only on product_id |
| **3NF** | 2NF + no transitive dependency (non-key column depends on another non-key column) | `zip_code → city` — city depends on zip, not on the primary key |

**Senior tip:** In practice, OLTP schemas aim for 3NF. OLAP/data warehouse schemas deliberately denormalize (star schema) for query performance.

---

### Q24. What is the difference between a star schema and a snowflake schema?

**Answer:**

| | Star Schema | Snowflake Schema |
|---|---|---|
| Dimension tables | Denormalized (flat) | Normalized (split into sub-tables) |
| Join complexity | Simple — one join per dimension | More joins needed |
| Query performance | Faster | Slower (more joins) |
| Storage | More (duplicate data) | Less |
| Use case | BI/reporting dashboards | When dimension data is huge and storage matters |

```
Star:  Sales → [Date, Customer, Product, Store]  (4 direct joins)
Snowflake: Sales → Product → Category → Department  (chain of joins)
```

---

### Q25. When would you use a surrogate key vs a natural key?

**Answer:**

| | Natural Key | Surrogate Key |
|---|---|---|
| Definition | A meaningful business value (email, SSN, ISBN) | A system-generated ID (AUTO_INCREMENT, UUID) |
| Pros | Self-documenting, no extra column | Stable, never changes, simple joins |
| Cons | Can change (email changes), composite keys slow joins | Meaningless without a lookup |
| When to use | When the business key is truly immutable and simple | Almost always — default choice |

**Senior tip:** Always use a surrogate PK and add a unique constraint on the natural key. The natural key can change (email changes); your PK should not.

---

### Q26. How would you handle a many-to-many relationship?

**Answer:** With a **junction table** (also called bridge/associative table).

```sql
-- Students can enroll in many courses; courses have many students
CREATE TABLE students  (id BIGINT PRIMARY KEY, name VARCHAR(100));
CREATE TABLE courses   (id BIGINT PRIMARY KEY, title VARCHAR(100));

-- Junction table — holds the relationship + any relationship attributes
CREATE TABLE enrollments (
    student_id  BIGINT REFERENCES students(id),
    course_id   BIGINT REFERENCES courses(id),
    enrolled_at DATE,
    grade       CHAR(2),
    PRIMARY KEY (student_id, course_id)
);
```

**Senior tip:** Index both FK columns in the junction table. The composite PK covers queries from one side; add a separate index for the other side:
```sql
CREATE INDEX idx_enroll_course ON enrollments (course_id);
```

---

## Advanced & Tricky

---

### Q27. What is the difference between DELETE, TRUNCATE, and DROP?

**Answer:**

| | `DELETE` | `TRUNCATE` | `DROP` |
|---|---|---|---|
| What it removes | Rows (all or filtered) | All rows | Entire table (structure + data) |
| WHERE clause | Yes | No | No |
| Transactional | Yes — can rollback | No (DDL) — cannot rollback | No (DDL) |
| Triggers fired | Yes | No | No |
| Auto-increment reset | No | Yes | Yes (table gone) |
| Speed | Slower (logs each row) | Fast (deallocates pages) | Instant |

---

### Q28. What is a CTE? How is it different from a subquery? What is a recursive CTE?

**Answer:** A CTE (`WITH` clause) is a named temporary result set. It improves readability and can be referenced multiple times.

```sql
-- Subquery: harder to read, can't reuse
SELECT name FROM (SELECT * FROM employees WHERE dept_id = 5) sub WHERE salary > 50000;

-- CTE: named, readable, reusable
WITH dept5_employees AS (
    SELECT * FROM employees WHERE dept_id = 5
)
SELECT name FROM dept5_employees WHERE salary > 50000;
```

**Recursive CTE** — for hierarchical data (org chart, category tree):

```sql
-- Walk the org hierarchy from CEO down
WITH RECURSIVE org_tree AS (
    -- Anchor: start with CEO (no manager)
    SELECT id, name, manager_id, 1 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: join employees to their manager already in the CTE
    SELECT e.id, e.name, e.manager_id, ot.level + 1
    FROM employees e
    INNER JOIN org_tree ot ON e.manager_id = ot.id
)
SELECT id, name, level FROM org_tree ORDER BY level;
```

---

### Q29. How would you find and remove duplicate rows from a table?

**Answer:**

```sql
-- Step 1: Find duplicates
SELECT email, COUNT(*) AS cnt
FROM users
GROUP BY email
HAVING COUNT(*) > 1;

-- Step 2: See which rows are duplicates (keep lowest id)
SELECT * FROM users
WHERE id NOT IN (
    SELECT MIN(id) FROM users GROUP BY email
);

-- Step 3: Delete duplicates (keep one row per email)
DELETE FROM users
WHERE id NOT IN (
    SELECT min_id FROM (
        SELECT MIN(id) AS min_id FROM users GROUP BY email
    ) AS keep
);

-- Modern approach using ROW_NUMBER (cleaner)
DELETE FROM users
WHERE id IN (
    SELECT id FROM (
        SELECT id, ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) AS rn
        FROM users
    ) ranked
    WHERE rn > 1
);
```

---

### Q30. What is the N+1 query problem? How do you solve it in SQL?

**Answer:** The N+1 problem occurs when you run 1 query to get N records, then run N additional queries to get related data for each — instead of fetching everything in one query.

```sql
-- BAD: 1 query to get all orders, then 1 query PER order to get customer name
-- Application code loop: for each order → SELECT * FROM customers WHERE id = ?
-- = 1 + N queries

-- GOOD: Join everything in one query
SELECT o.id, o.total, c.name AS customer_name
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id;

-- GOOD: For multiple items per order, use GROUP_CONCAT or a lateral join
SELECT
    o.id,
    o.total,
    c.name AS customer_name,
    GROUP_CONCAT(p.name ORDER BY p.name) AS products
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id
INNER JOIN order_items oi ON oi.order_id = o.id
INNER JOIN products p ON p.id = oi.product_id
GROUP BY o.id, o.total, c.name;
```

**Senior tip:** In ORMs (Hibernate, JPA), N+1 is triggered by lazy loading inside a loop. Fix with `JOIN FETCH` in JPQL or `@EntityGraph`. Always check SQL logs in dev — if you see hundreds of identical queries with different IDs, you have N+1.

---

## Quick Reference Summary

| Topic | Key Questions |
|---|---|
| Joins | Q1 (join types), Q2 (self join), Q5 (WHERE vs HAVING) |
| Subqueries | Q3 (EXISTS vs IN), Q4 (correlated), Q6 (Nth highest) |
| Window Functions | Q7 (vs GROUP BY), Q8 (RANK types), Q9 (LAG/LEAD), Q11 (top-N per group) |
| Performance | Q13 (indexes), Q14 (composite/leftmost prefix), Q16 (EXPLAIN), Q18 (slow queries) |
| Transactions | Q19 (ACID), Q20 (isolation levels), Q21 (deadlock), Q22 (optimistic vs pessimistic) |
| Schema Design | Q23 (normal forms), Q25 (surrogate vs natural key), Q26 (many-to-many) |
| Advanced | Q27 (DELETE vs TRUNCATE), Q28 (recursive CTE), Q29 (remove duplicates), Q30 (N+1) |
