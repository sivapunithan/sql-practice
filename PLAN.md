# Informatica / SQL Interview Prep Plan

**Interview date:** TBD
**Plan created:** 2026-11-01
**Target window:** ~2 weeks (compressed pace)
**Daily block:** ~1 hr SQL exercises + ~20-30 min Informatica concept study
**DB:** Pagila (`sqlpractice`, via `docker exec -it sql-practice-pg psql -U practice -d sqlpractice`)

## Rules
- No full solutions written here — problem + pattern/formula only. You write the actual query.
- Never modify `pagila`/`sqlpractice` core tables. New tables go in a separate `etl_practice` schema, created only after explicit OK.
- Any file/DB write Claude proposes gets confirmed before it happens.

## How to log progress
After each exercise, add a row to its **Log** table: date, time spent, status (`attempted` / `stuck` / `done`), and a one-line note (what broke, what you learned). Update the **Status** line at the top of each day's section (`not started` / `in progress` / `done`).

---

## Day 1 — Joins (all types, row-count effects)
**Status:** done (2026-11-01)

### 1.1 Inner vs Left join row count
**Tables:** `film`, `inventory`
**Problem:** Compare row counts of `film INNER JOIN inventory` vs `film LEFT JOIN inventory`. Find films never stocked at any store.
**Pattern:**
```sql
SELECT ...
FROM a
[INNER|LEFT] JOIN b ON a.key = b.key
```
LEFT keeps all of `a`; unmatched `b` columns come back NULL. Films with no inventory row = the ones where the LEFT join's `inventory` columns are NULL.
**Check:** LEFT join count ≥ INNER join count. The unmatched rows (anti-join) should be the diff.
**Oracle note:** Oracle supports legacy `(+)` outer-join syntax (e.g. `WHERE a.key = b.key(+)`) — you may see it in old code. Know it maps to LEFT/RIGHT JOIN, but ANSI `JOIN` is the standard to write.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|
| 2026-11-01 | | done | INNER=4581, LEFT=4623, 42 unstocked films isolated via `i.film_id IS NULL`. Verified count. |
| | | | |

### 1.2 Join chain (1:1 / many:1)
**Tables:** `customer → address → city → country`
**Problem:** Get each customer's country.
**Pattern:** chain multiple `JOIN`s, each on the FK → PK of the next table.
**Check:** result row count should equal `customer` count (999) if every customer resolves to a country — verify, don't assume.
**Oracle note:** none specific; standard ANSI join chaining works the same.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|
| 2026-11-01 | | done | 4-table chain customer→address→city→country, 999 rows = exact customer count. |

### 1.3 Many-to-many via bridge table
**Tables:** `film`, `film_actor`, `actor`
**Problem:** Count rows before/after joining through the bridge table; explain why the joined row count isn't `film` count or `actor` count.
**Pattern:** `a JOIN bridge ON a.pk = bridge.fk1 JOIN b ON bridge.fk2 = b.pk` — row count driven by the bridge table.
**Check:** total rows should equal `film_actor` row count (5,462) when joining all three.
**Informatica mapping:** Joiner transformation — Normal / Master Outer / Detail Outer / Full Outer join types map to INNER / RIGHT / LEFT / FULL SQL joins.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|
| 2026-11-01 | | done | 5462 rows = film_actor's own row count, not film or actor count. |

---

## Day 2 — Anti-joins & duplicates
**Status:** not started

### 2.1 Anti-join: customers with zero rentals
**Tables:** `customer`, `rental`
**Problem:** Find customers who have never rented anything.
**Pattern:**
```sql
-- option 1
SELECT a.* FROM a LEFT JOIN b ON a.key = b.key WHERE b.key IS NULL;
-- option 2 (equivalent, often faster, and safe with NULLs)
SELECT a.* FROM a WHERE NOT EXISTS (SELECT 1 FROM b WHERE b.key = a.key);
```
**Trap:** never use `NOT IN (SELECT key FROM b)` if `b.key` can contain NULL — it silently returns zero rows for the whole query.
**Check:** write it both ways (LEFT JOIN / NOT EXISTS) and confirm identical result sets.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

### 2.2 Anti-join: unstocked films
**Tables:** `film`, `inventory`
**Problem:** Films with no inventory row at any store.
**Pattern:** same anti-join pattern as 2.1.
**Check:** cross-verify count against Day 1.1's diff — should match exactly.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

### 2.3 Find duplicate customers
**Table:** `customer`
**Problem:** Find any `(first_name, last_name)` or `email` combos appearing more than once.
**Pattern:**
```sql
SELECT col1, col2, COUNT(*)
FROM t
GROUP BY col1, col2
HAVING COUNT(*) > 1;
```
**Check:** if zero rows returned, explicitly verify there truly are no dupes (don't assume empty = correct).
**Informatica mapping:** anti-join ≈ Lookup transformation configured to return unmatched rows, or a Joiner + filter on NULL. Duplicate detection ≈ Aggregator with COUNT, or Sorter + Expression flagging consecutive dupes.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

---

## Day 3 — GROUP BY / HAVING
**Status:** not started

### 3.1 Aggregate per group
**Table:** `payment`
**Problem:** Total revenue per `customer_id`.
**Pattern:**
```sql
SELECT group_col, AGG(col)
FROM t
[WHERE row-level filter]
GROUP BY group_col
HAVING AGG(col) condition;
```
Rule: `WHERE` filters rows before grouping; `HAVING` filters groups after aggregation. No aggregate allowed in `WHERE`.
**Check:** sum of all group totals should equal grand total `SUM(amount)` from the whole table.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

### 3.2 Filter groups
**Table:** `rental`
**Problem:** Staff members who processed more than 5,000 rentals.
**Pattern:** same as 3.1, with `HAVING COUNT(*) > n`.
**Check:** sanity-check against total `rental` row count ÷ number of staff.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

### 3.3 Group by time bucket
**Table:** `payment`
**Problem:** Monthly revenue trend.
**Pattern (Postgres):** `GROUP BY DATE_TRUNC('month', payment_date)`
**Oracle note:** `DATE_TRUNC` doesn't exist in Oracle — use `TRUNC(payment_date, 'MM')`.
**Check:** sum across all months = grand total.
**Informatica mapping:** Aggregator transformation — GROUP BY columns = group-by ports; HAVING = filter condition after aggregation, or a downstream Filter transformation.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

---

## Day 4 — Self-joins
**Status:** not started

### 4.1 Hierarchy self-join
**Tables:** `store`, `staff`
**Problem:** Join `store.manager_staff_id` back to `staff.staff_id` to get manager names per store.
**Pattern:**
```sql
SELECT x.*, y.*
FROM t x
JOIN t y ON x.fk_col = y.pk_col;
```
**Check:** row count should equal `store` count (500) unless some stores have no manager set — verify.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

### 4.2 Pairwise comparison
**Table:** `customer`
**Problem:** Find pairs of customers sharing the same `address_id`.
**Pattern:**
```sql
SELECT x.*, y.*
FROM t x
JOIN t y ON x.shared_col = y.shared_col AND x.id < y.id;
```
The `x.id < y.id` avoids getting both (A,B) and (B,A).
**Check:** confirm no pair appears twice.
**Informatica mapping:** self-join is usually modeled as two instances of the same Source Qualifier/Lookup against the same table, joined via a Joiner or Lookup condition.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

---

## Day 5 — Subqueries & CTEs
**Status:** not started

### 5.1 Correlated subquery
**Tables:** `film`, `film_category`
**Problem:** Films priced above the average `rental_rate` of their own category.
**Pattern:**
```sql
SELECT * FROM a
WHERE a.col > (SELECT AVG(col) FROM b WHERE b.group_key = a.group_key);
```
**Check:** result count should be under total film count, roughly under half.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

### 5.2 Same thing as CTE
**Problem:** Rewrite 5.1 using `WITH`.
**Pattern:**
```sql
WITH group_avg AS (
  SELECT group_key, AVG(col) AS avg_col FROM b GROUP BY group_key
)
SELECT a.* FROM a
JOIN group_avg g ON a.group_key = g.group_key
WHERE a.col > g.avg_col;
```
**Check:** result sets from 5.1 and 5.2 must match exactly. Optionally compare `EXPLAIN` output.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

### 5.3 Multi-step CTE
**Tables:** `payment`, `customer`
**Problem:** Customer lifetime value — CTE computing per-customer total, then filter top 10% by value.
**Pattern:** chain two `WITH` blocks, second referencing the first.
**Check:** sanity-check the top customer's total against a manual `ORDER BY total DESC LIMIT 1` query.
**Informatica mapping:** CTEs/subqueries usually flatten into a Source Qualifier custom SQL override, or become a Lookup transformation (correlated subquery ≈ connected Lookup per row) or an Aggregator-then-Joiner pipeline (CTE with aggregation).
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

---

## Day 6 — Window functions I (ranking)
**Status:** not started

### 6.1 ROW_NUMBER basics
**Table:** `payment`
**Problem:** Number each customer's payments in date order.
**Pattern:**
```sql
SELECT *, ROW_NUMBER() OVER (PARTITION BY group_col ORDER BY sort_col) AS rn
FROM t;
```
**Check:** max `rn` per customer should equal that customer's payment count from a separate `GROUP BY COUNT(*)` query.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

### 6.2 RANK vs DENSE_RANK vs ROW_NUMBER
**Table:** `film`
**Problem:** Rank films by `rental_rate` within each `rating`, using all three functions side by side; find a tie to see them diverge.
**Pattern:**
```sql
SELECT *,
  ROW_NUMBER() OVER (PARTITION BY group_col ORDER BY val DESC) AS rn,
  RANK()       OVER (PARTITION BY group_col ORDER BY val DESC) AS rnk,
  DENSE_RANK() OVER (PARTITION BY group_col ORDER BY val DESC) AS drnk
FROM t;
```
`ROW_NUMBER`: always unique, no ties. `RANK`: ties share a rank, next rank skips (1,1,3). `DENSE_RANK`: ties share a rank, no skip (1,1,2).
**Check:** pick a `rating` group with a known tie in `rental_rate`, confirm RANK skips and DENSE_RANK doesn't.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

### 6.3 Nth highest per group
**Table:** `payment`
**Problem:** 2nd-highest single payment amount per customer.
**Pattern:**
```sql
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY group_col ORDER BY val DESC) AS rn
  FROM t
) sub
WHERE rn = N;
```
**Check:** manually verify for one `customer_id` using `ORDER BY amount DESC LIMIT 2`.
**Informatica mapping:** ranking functions ≈ Rank transformation (Top/Bottom N per group using a "Group By" port). No direct equivalent for DENSE_RANK — usually needs an Expression + variable-port workaround.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

---

## Day 7 — Window functions II (LAG/LEAD, running totals) + SCD2 intro
**Status:** not started

### 7.1 LAG/LEAD
**Table:** `payment`
**Problem:** For each customer, get the gap in days between consecutive payments.
**Pattern:**
```sql
SELECT *,
  LAG(col)  OVER (PARTITION BY group_col ORDER BY date_col) AS prev_val,
  LEAD(col) OVER (PARTITION BY group_col ORDER BY date_col) AS next_val
FROM t;
```
**Check:** first payment per customer should have a NULL `prev_val`.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

### 7.2 Running total
**Table:** `payment`
**Problem:** Running cumulative revenue per store ordered by date.
**Pattern:**
```sql
SELECT *,
  SUM(col) OVER (PARTITION BY group_col ORDER BY date_col
                  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM t;
```
**Check:** final running total per store should equal that store's `SUM(amount)`.
**Informatica mapping:** LAG/LEAD ≈ no direct transformation — typically sorted data + Expression transformation with variable ports holding the "previous row" value. Running totals ≈ Expression with a cumulative variable port.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

### 7.3 SCD Type 2 — concept only (no query yet)
**Problem:** Given `customer.last_update`, think through (pseudocode/English only) how you'd detect a changed row and version it (`effective_date`, `end_date`, `is_current` flag). Day 9 builds it for real.
**Check:** n/a — this is a design exercise.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

---

## Day 8 — MERGE-style upserts
**Status:** not started

### 8.1 Propose `etl_practice` schema — WAIT FOR OK
Needed: a small staging table mimicking incoming `customer` changes (a few updated rows + a few new rows) to practice upsert logic against. **Do not create until explicitly approved.**
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

### 8.2 Upsert
**Problem:** Upsert staging-table rows into a target table.
**Pattern (Postgres):**
```sql
INSERT INTO target (key_col, col1, col2)
VALUES (...)
ON CONFLICT (key_col)
DO UPDATE SET col1 = EXCLUDED.col1, col2 = EXCLUDED.col2;
```
**Oracle equivalent (memorize — high interview likelihood):**
```sql
MERGE INTO target t
USING source s ON (t.key_col = s.key_col)
WHEN MATCHED THEN UPDATE SET t.col1 = s.col1
WHEN NOT MATCHED THEN INSERT (key_col, col1) VALUES (s.key_col, s.col1);
```
**Check:** target row count after upsert = original target rows + genuinely new rows (updates shouldn't add rows).
**Informatica mapping:** Update Strategy transformation (flags rows DD_INSERT/DD_UPDATE/DD_DELETE) combined with a Lookup to check existence — this is literally how Informatica implements upserts.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

---

## Day 9 — SCD Type 2 (build)
**Status:** not started

### 9.1 Build versioned dimension
**Schema:** `etl_practice` (already approved in Day 8)
**Problem:** Build a versioned `dim_customer` table (`customer_key` surrogate, `customer_id` natural key, tracked attributes, `effective_date`, `end_date`, `is_current`). Simulate one customer's address or email changing. Write the query that closes out the old version and inserts the new one.
**Pattern:**
```sql
-- step 1: expire the changed current row
UPDATE dim
SET end_date = CURRENT_DATE, is_current = false
WHERE natural_key = :id AND is_current = true
  AND (tracked_col1 IS DISTINCT FROM :new_val1 OR ...);

-- step 2: insert new version
INSERT INTO dim (natural_key, tracked_col1, effective_date, end_date, is_current)
VALUES (:id, :new_val1, CURRENT_DATE, NULL, true);
```
Use `IS DISTINCT FROM` to detect change — handles NULLs correctly, unlike `!=`.
**Oracle note:** `IS DISTINCT FROM` doesn't exist pre-23c. Workaround: `DECODE(a, b, 0, 1) = 1` or `NVL(a,'~') != NVL(b,'~')`.
**Check:** for any `customer_id`, exactly one row with `is_current = true`; no gaps/overlaps between `effective_date`/`end_date` across versions.
**Informatica mapping:** the classic SCD Type 2 mapping template — Lookup (find existing current row) → Expression (compare source vs lookup to detect change) → Router (new vs changed vs unchanged) → Update Strategy (expire old row) + separate insert path (new row, new surrogate key).
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

---

## Day 10 — ETL: incremental load
**Status:** not started

### 10.1 Incremental extract
**Tables:** any with `last_update` (e.g. `customer`, `film`)
**Problem:** Pull only rows changed since the last run.
**Pattern:**
```sql
SELECT * FROM source
WHERE last_update > :last_run_timestamp;
```
Next run's `:last_run_timestamp` = `MAX(last_update)` from what you just pulled (or `CURRENT_TIMESTAMP` at run start — note the clock-skew edge case).
**Check:** result is a strict subset of full table count; re-running with yesterday's already-processed timestamp should return 0 rows.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

### 10.2 Incremental load without `last_update`
**Tables:** `payment`, `rental`
**Problem:** These tables don't have `last_update` — they use `payment_date`/`rental_date` instead. Write the incremental logic using those columns, and in your own notes explain why using an immutable "created" timestamp vs a mutable "last modified" timestamp changes correctness (does it catch updates, not just inserts?).
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

**Informatica mapping:** incremental load ≈ Source Qualifier with a parameterized WHERE clause, commonly driven by a mapping variable `$$LastRunDate` updated via `SetMaxVariable`.

---

## Day 11 — ETL: reconciliation & data validation
**Status:** not started

### 11.1 Row-count reconciliation
**Tables:** `rental`, `payment`
**Problem:** Compare counts — they won't match exactly. Investigate and quantify why (not every rental has a payment).
**Pattern:**
```sql
SELECT 'source' src, COUNT(*) cnt, SUM(amount) total FROM src_table
UNION ALL
SELECT 'target', COUNT(*), SUM(amount) FROM tgt_table;
```
**Check:** quantify the gap and explain it — don't just note the numbers differ.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

### 11.2 Total reconciliation
**Problem:** Sum `payment.amount` grouped by `customer_id` vs. an expected "customer total spend" figure — check for double counting from a bad join.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

### 11.3 Missing-key validation
**Tables:** `payment`/`customer`, `rental`/`inventory`
**Problem:** Verify every `payment.customer_id` exists in `customer`, every `rental.inventory_id` exists in `inventory` — worth checking since `payment` has no FK constraint in this DB.
**Pattern:** anti-join pattern from 2.1.
**Check:** expect 0 orphans; if not, that's your finding — double-check your query logic first before concluding it's a data bug.
**Informatica mapping:** reconciliation/validation ≈ a separate control/audit mapping — Aggregator comparing source vs target row counts and sums, often written to an audit table. Missing-key checks ≈ unconnected Lookup returning NULL, flagged downstream.
**Log:**
| Date | Time | Status | Note |
|---|---|---|---|

---

## Days 12-13 — Review + weak-spot repeat
**Status:** not started
Re-do 2-3 of your weakest exercises from memory, no notes. Re-read Informatica notes on Lookup (connected vs unconnected, cache types) and Update Strategy — these come up constantly independent of the SQL questions.
**Log:**
| Date | Time | Topic repeated | Result |
|---|---|---|---|

## Day 14 — Buffer / mock run-through
**Status:** not started
Pick 3 random exercises across different days, solve cold, time yourself.
**Log:**
| Date | Exercise | Time taken | Result |
|---|---|---|---|

---

## Open items
- [ ] Interview date not yet confirmed — update top of file once known.
- [ ] `etl_practice` schema not yet created — needs OK before Day 8.
