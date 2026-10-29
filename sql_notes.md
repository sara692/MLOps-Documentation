# SQL — Commands & Concepts Summary

Reference built from hands-on queries against the **Chinook** database (a sample music-store database with tables like `Artist`, `Album`, `Track`, `Customer`, `Invoice`, `Employee`, `Genre`, `Playlist`).

---

## 1. Basic SELECT & Filtering

```sql
SELECT * FROM Album LIMIT 5;
```
- **SELECT** — retrieves data from a table. `*` means "all columns." **LIMIT** caps how many rows come back — good practice when previewing a large table.

```sql
SELECT FirstName, LastName, Country FROM Customer
ORDER BY Country;
```
- **ORDER BY** — sorts results by one or more columns. Defaults to ascending (`ASC`); add `DESC` for descending.

```sql
SELECT * FROM Track
WHERE UnitPrice > 0.99;
```
- **WHERE** — filters rows *before* grouping, based on a condition.

```sql
SELECT * FROM Customer
WHERE Country IS 'Brazil';
```
- **IS** — used here like `=` for an exact match (commonly `IS` is reserved for comparing against `NULL`, e.g. `WHERE Country IS NULL`; using it for a literal string works in SQLite but `=` is the more standard choice).

```sql
SELECT * FROM Employee
WHERE Title LIKE '%Manager%';
```
- **LIKE** — pattern matching for text. `%` = any number of characters (wildcard), `_` = exactly one character. Here it finds any title containing the word "Manager."

```sql
SELECT DISTINCT(Country) FROM Customer;
```
- **DISTINCT** — removes duplicate values, returning only unique entries.

---

## 2. Aggregate Functions

```sql
SELECT COUNT(TrackId) FROM Track;
```
- **COUNT** — counts the number of rows (or non-null values in a column).

```sql
SELECT Avg(UnitPrice) FROM Track;
```
- **AVG** — calculates the mean of a numeric column.

```sql
SELECT SUM(I.Total), I.BillingCountry FROM Invoice AS I
GROUP BY BillingCountry;
```
- **SUM** — adds up all values in a column, typically combined with `GROUP BY` (see §3) to get subtotals per category.

> Other common aggregates: `MIN()`, `MAX()` — smallest/largest value in a column.

---

## 3. GROUP BY & HAVING

```sql
SELECT G.Name, COUNT(T.TrackId) FROM Genre AS G
LEFT JOIN Track AS T ON G.GenreId = T.GenreId
GROUP BY G.Name
ORDER BY COUNT(T.TrackId) DESC;
```
- **GROUP BY** — collapses rows sharing the same value in a column into a single summary row, so aggregate functions (COUNT, SUM, AVG) apply *per group* rather than to the whole table.

```sql
SELECT G.Name, COUNT(T.TrackId)
FROM Genre AS G
LEFT JOIN Track AS T ON G.GenreId = T.GenreId
GROUP BY G.Name
HAVING COUNT(T.TrackId) > 100;
```
- **HAVING** — filters groups *after* aggregation (unlike `WHERE`, which filters individual rows *before* grouping). Use `HAVING` when the condition involves an aggregate function like `COUNT()` or `SUM()`.

---

## 4. Joins

```sql
SELECT * FROM Artist
RIGHT JOIN Album ON Artist.ArtistId = Album.ArtistId
LIMIT 10;
```
- **RIGHT JOIN** — keeps all rows from the right-hand table (`Album`), matching in rows from the left table (`Artist`) where possible, filling with `NULL` where there's no match. (Note: SQLite doesn't natively support `RIGHT JOIN` in older versions — it's included here as it appeared in the source notebook, but a `LEFT JOIN` with tables swapped is the portable equivalent.)

```sql
SELECT A.Title, R.Name FROM Album AS A
LEFT JOIN Artist AS R ON A.ArtistId = R.ArtistId;
```
- **LEFT JOIN** — keeps all rows from the left table (`Album`), matching in rows from the right table (`Artist`) where possible. The most commonly used join — "give me everything from A, plus matching info from B if it exists."

```sql
SELECT Track.Name, SUM(InvoiceLine.Quantity) AS TotalSold
FROM InvoiceLine
JOIN Track ON InvoiceLine.TrackId = Track.TrackId
GROUP BY Track.TrackId;
```
- **JOIN** (a.k.a. `INNER JOIN`) — only keeps rows that have a match in **both** tables. Rows with no match on either side are excluded entirely.

```sql
SELECT E.FirstName AS Employee_FirstName, P.FirstName AS Manager_FirstName
FROM Employee AS E
LEFT JOIN Employee AS P ON E.ReportsTo = P.EmployeeId;
```
- **Self join** — a table joined to itself (using two aliases), useful for hierarchical relationships, like an employee's manager also being a row in the same `Employee` table.

> `AS` creates an **alias** (shorthand name) for a table or column — used heavily above (`Employee AS E`) to keep long queries with multiple joins readable.

---

## 5. Subqueries

```sql
SELECT * FROM Track
WHERE UnitPrice > (SELECT Avg(UnitPrice) FROM Track);
```
- **Subquery** — a query nested inside another query. Here, the inner query calculates the average price first, and the outer query filters tracks against that value.

```sql
SELECT Customer.CustomerId, Customer.FirstName, Customer.LastName
FROM Customer
LEFT JOIN Invoice ON Customer.CustomerId = Invoice.CustomerId
WHERE Invoice.InvoiceId IS NULL;
```
- **Finding "no match" rows** — a `LEFT JOIN` combined with `WHERE ... IS NULL` finds rows in the left table that have **no** corresponding match in the right table (e.g. customers who've never placed an invoice).

```sql
SELECT C.FirstName || " " || C.LastName,
       (SELECT MAX(InvoiceDate) FROM Invoice AS I WHERE C.CustomerId = I.CustomerId) AS Latest_Date
FROM Customer AS C
ORDER BY Latest_Date DESC;
```
- **Correlated subquery** — an inner query that references a column from the outer query (`C.CustomerId` here), so it re-runs once per outer row instead of just once overall. More flexible, but generally slower than an equivalent `JOIN`.
- `||` — string concatenation operator in SQLite (joins `FirstName` and `LastName` with a space between them).

---

## 6. CASE — Conditional Logic

```sql
SELECT COUNT(T.TrackId),
CASE
    WHEN T.Milliseconds/(1000*60) < 3 THEN 'Short'
    WHEN T.Milliseconds/(1000*60) BETWEEN 3 AND 6 THEN 'Medium'
    ELSE 'Long'
END AS Length_Category
FROM Track AS T
GROUP BY Length_Category;
```
- **CASE WHEN** — SQL's version of if/else logic, used to bucket values into custom categories on the fly. Evaluates each `WHEN` condition top to bottom, uses `ELSE` if none match.
- **BETWEEN** — shorthand for "greater than or equal to X and less than or equal to Y."

---

## 7. CTEs (Common Table Expressions) — `WITH`

```sql
WITH CustomerSpending AS (
    SELECT C.CustomerId, C.FirstName, C.LastName, C.Country,
           COALESCE(SUM(I.Total), 0) AS TotalSpent,
           COUNT(I.InvoiceId) AS InvoiceCount
    FROM Customer C
    LEFT JOIN Invoice I ON C.CustomerId = I.CustomerId
    GROUP BY C.CustomerId, C.FirstName, C.LastName, C.Country
)
SELECT Country, FirstName, LastName, TotalSpent,
       RANK() OVER (PARTITION BY Country ORDER BY TotalSpent DESC) AS CountrySpendRank
FROM CustomerSpending
ORDER BY Country, CountrySpendRank;
```
- **WITH (CTE)** — defines a named, temporary result set (`CustomerSpending` here) that can be referenced later in the same query, just like a table. Makes complex multi-step queries far more readable than nesting subqueries inside subqueries.
- **COALESCE** — returns the first non-`NULL` value in a list; here it replaces `NULL` totals (customers with no invoices) with `0` instead.

---

## 8. Window Functions

Window functions calculate something **across a set of rows related to the current row**, without collapsing the results into a single row per group (unlike `GROUP BY`).

```sql
SELECT CustomerId, Country, Total,
       RANK() OVER (PARTITION BY Country ORDER BY Total) AS Rank
FROM Customer_Invoice;
```
- **RANK() OVER (...)** — assigns a rank number to each row within a group. `PARTITION BY` defines the group (restarts ranking per country here); `ORDER BY` decides the ranking order. Ties get the same rank, and the next rank skips accordingly (e.g. 1, 2, 2, 4).

```sql
SELECT ArtistName, AlbumTitle, Revenue,
       ROW_NUMBER() OVER (PARTITION BY ArtistId ORDER BY Revenue DESC) AS rn
FROM AlbumRevenue;
```
- **ROW_NUMBER() OVER (...)** — same idea as `RANK()`, but always assigns a strictly increasing unique number with no ties/gaps (1, 2, 3, 4...), even if values are equal.

```sql
SELECT InvoiceDate, Total,
       SUM(Total) OVER (ORDER BY InvoiceDate) AS RunningTotal
FROM Invoice;
```
- **Running total** — `SUM() OVER (ORDER BY ...)` without `PARTITION BY` accumulates a total across all preceding rows in the specified order — a classic use of window functions for cumulative metrics.

```sql
WITH MONTH_REVENUE AS (
    SELECT strftime('%Y-%m', InvoiceDate) AS Month_Date, SUM(Total) AS REVENUE_PER_MONTH
    FROM Invoice GROUP BY Month_Date
)
SELECT Month_Date, REVENUE_PER_MONTH,
       LAG(REVENUE_PER_MONTH) OVER (ORDER BY Month_Date) AS PREVIOUS_MONTH_REVENUE
FROM MONTH_REVENUE;
```
- **LAG()** — pulls the value from a **previous** row (commonly used to compare a value to the one before it, e.g. month-over-month change). **LEAD()** does the same but looks **forward** to a following row instead.
- **strftime** — SQLite's date-formatting function; `'%Y-%m'` extracts just year-month from a full date (used here to group revenue by month).

> **Window function vs. GROUP BY:** `GROUP BY` collapses rows into one summary row per group. Window functions (`OVER (...)`) keep every original row, just adding a calculated column alongside it — so you can see individual rows *and* their rank/running total/group context together.

---

## 9. Creating & Modifying Tables

```sql
CREATE TABLE IF NOT EXISTS Country(
    CountryId INTEGER PRIMARY KEY,
    CountryName TEXT NOT NULL UNIQUE
);
```
- **CREATE TABLE** — defines a new table and its columns. `IF NOT EXISTS` prevents an error if the table's already there.
- **PRIMARY KEY** — uniquely identifies each row in the table; can't be `NULL` or duplicated.
- **NOT NULL** — this column must always have a value.
- **UNIQUE** — no two rows can share the same value in this column.

```sql
CREATE TABLE IF NOT EXISTS Employee_Backup AS SELECT * FROM Employee;
```
- **CREATE TABLE ... AS SELECT** — creates a new table pre-filled with the results of a query — a quick way to snapshot/back up data or save a complex query's results as a reusable table.

```sql
CREATE TABLE IF NOT EXISTS WishList(
    WishlistId INTEGER PRIMARY KEY,
    TrackId INTEGER,
    FOREIGN KEY (TrackId) REFERENCES Track(TrackId)
);
```
- **FOREIGN KEY** — links a column in this table to the primary key of another table (`Track.TrackId` here), enforcing that any `TrackId` inserted here must actually exist in the `Track` table. Maintains relational integrity between tables.

---

## 10. Inserting Data

```sql
INSERT INTO Country VALUES(1, 'Egypt');
```
- **INSERT INTO ... VALUES** — adds a new row. Values must be listed in the same order as the table's columns (or you can specify column names explicitly, as below).

```sql
INSERT INTO Wishlist (WishlistId, TrackId) VALUES (1, 2);
```
- Same as above, but naming the columns explicitly — safer, since it doesn't rely on remembering the table's exact column order.

```sql
PRAGMA foreign_keys = ON;
```
- **PRAGMA** — a SQLite-specific command for configuring database behavior. `foreign_keys = ON` enables foreign key constraint enforcement (SQLite has this **off by default**, unlike most other databases — without this, foreign key rules defined in `CREATE TABLE` are silently ignored).

---

## Quick Reference — Cheat Sheet

| Clause / Function | Purpose |
|---|---|
| `SELECT ... FROM` | Retrieve columns from a table |
| `WHERE` | Filter rows before grouping |
| `GROUP BY` | Collapse rows into per-group summaries |
| `HAVING` | Filter groups after aggregation |
| `ORDER BY` | Sort results |
| `LIMIT` | Cap number of rows returned |
| `DISTINCT` | Remove duplicate rows |
| `LIKE '%text%'` | Pattern match text |
| `JOIN` / `INNER JOIN` | Rows matching in both tables |
| `LEFT JOIN` | All rows from left table + matches from right |
| `RIGHT JOIN` | All rows from right table + matches from left |
| `AS` | Alias a table or column |
| `COUNT / SUM / AVG / MIN / MAX` | Aggregate functions |
| `CASE WHEN ... THEN ... ELSE ... END` | Conditional logic |
| `WITH name AS (...)` | Common Table Expression (named subquery) |
| `RANK() OVER (...)` | Rank rows within a group (ties share rank, gaps after) |
| `ROW_NUMBER() OVER (...)` | Unique sequential number per row |
| `LAG() / LEAD()` | Access a previous / following row's value |
| `SUM(...) OVER (ORDER BY ...)` | Running/cumulative total |
| `COALESCE(x, default)` | Replace `NULL` with a fallback value |
| `CREATE TABLE` | Define a new table |
| `PRIMARY KEY` | Unique row identifier |
| `FOREIGN KEY ... REFERENCES` | Link to another table's primary key |
| `INSERT INTO ... VALUES` | Add a new row |
| `PRAGMA foreign_keys = ON` | Enable foreign key enforcement (SQLite-specific) |
