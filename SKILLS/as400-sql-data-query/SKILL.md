---
name: as400-sql-data-query
description: Answer questions about the data stored in IBM i files (DB2 for i physical and logical files, tables, views) by running read-only SELECT/WITH queries with the as400_sql_query tool. Use for lists, counts, aggregates, joins and checks on real data, including legacy ERP files with cryptic names. Explains the guard rules (single statement, no DML, qualified table names), IBM i SQL dialect points and how to present results with their query and assumptions.
---

# Querying data with `as400_sql_query`

Prerequisites: `as400-mcp-basics`; use `as400-database-catalog` to find the right files and fields first.

## The tool

`as400_sql_query(query, parameters?)`: runs one **SELECT** or **WITH** statement and returns `rows` and `data`
(array of objects keyed by column name; `data` is never translated).

Guard rules (enforced by the server; violating them returns an error, never a partial effect):
- Only `SELECT`/`WITH`. No `INSERT/UPDATE/DELETE/MERGE/CALL/CREATE/DROP/...`, no `FOR UPDATE`, no `FINAL TABLE`.
- **One statement, no trailing semicolon.** Comments and strings must be closed.
- `QCMDEXC` and the IFS/spool table functions (`IFS_READ`, `SPOOLED_FILE_DATA`, ...) are blocked here: use the dedicated tools.
- If `ALLOWED_LIBS` is active, **every table must be qualified `LIB.TABLE`** with an allowed library (catalogs `QSYS2`,
  `SYSIBM`, `SYSIBMADM`, `SYSTOOLS` are always readable). Qualify names anyway: it avoids library-list surprises.
- The connection is read-only at the database level, so writes would be rejected by IBM i itself as well.
- Row cap `SQL_MAX_ROWS` is applied automatically and signaled in the result; timeout 60 s per query.

## Workflow for a question like "list the customers of Milan"

1. **Find the file and fields**: `as400_table_list` (description `text`), `as400_column_search`, `as400_file_desc`.
2. **Peek at the data** with `FETCH FIRST 5 ROWS ONLY` to see formats (padding, case, codes, dates, decimals).
3. **Discover the real values** of coded fields: `SELECT DISTINCT FIELD FROM LIB.FILE FETCH FIRST 50 ROWS ONLY`. Do not
   assume that a province is `'MI'`, `'Milano'` or `'MILANO'`; do not assume what a flag means.
4. **Write the final query** with the minimum columns, explicit `WHERE`, and a sensible `ORDER BY` and row limit.
5. **Run, then verify** (count matches expectations? any NULL/blank anomalies? duplicates from a join?).
6. **Answer**: result table (trimmed to what the user needs), the **query**, the **assumptions** (which file/fields,
   how codes were interpreted, filters applied) and the **limits** (rows capped? files that might also hold the answer?).

## IBM i SQL notes

- Use `FETCH FIRST n ROWS ONLY` (not `LIMIT`). `LOWER`, `UPPER`, `TRIM`, `SUBSTR`, `LIKE`, `COALESCE`, `CASE`,
  `CAST(x AS VARCHAR(20))`, `DIGITS(n)`, `INT`, `DECIMAL`, `CURRENT DATE`, date arithmetic with `+ 1 DAY` all work.
- **CHAR columns are blank-padded**: compare with `TRIM(col) = 'X'` or `col = 'X'` (padding is ignored in comparisons, but
  not in concatenation or `LIKE` with fixed patterns). Search case-insensitively with `UPPER(col) LIKE '%MILAN%'`.
- **Dates in legacy files** are often numbers (`20240131`) or `CYYMMDD` (`1240131`), or 0 meaning "none": convert with
  `DATE(TIMESTAMP_FORMAT(CHAR(col), 'YYYYMMDD'))` only after checking for zeros; validate with `MIN`/`MAX` first.
- **Numbers**: packed/zoned decimals respect declared scale; the tool returns numbers as strings.
- **File names**: `LIB.FILE` for a PF/LF/table. For a **logical file** you query the LF name like a table (it applies its
  select/omit and key order). Multi-member files read the first member by default.
- Qualify catalog queries: `QSYS2.SYSTABLES`, `QSYS2.SYSCOLUMNS`, `QSYS2.SYSPARTITIONSTAT` (row counts per member:
  `NUMBER_ROWS`), `QSYS2.SYSKEYS`, `QSYS2.SYSINDEXES`. Always filter by schema and name; a `GROUP BY` over a whole big
  library can time out. Use the dedicated catalog tools when they exist.
- Parameter markers: `WHERE COL = ?` with `parameters: ["X"]` (strings, numbers, null).
- `WITH` (CTE) is allowed; so are subselects, `UNION`, window functions (`ROW_NUMBER() OVER ...`) and `LISTAGG`.
- Counting: `SELECT COUNT(*) FROM LIB.FILE` is cheap; `COUNT(DISTINCT x)` on very large files may be slow. Add `WHERE`
  filters on keyed fields to benefit from access paths.

## Joins on legacy models

No foreign keys usually exist. Join on key fields with matching type and length. After the first join check cardinality
(`COUNT(*)` vs `COUNT(DISTINCT key)`) to detect accidental multiplication. For "customers of a province" the address may be
in a separate table from the customer master: confirm which table holds the *current* address.

## Safety and privacy

- Select only the columns needed; avoid dumping personal data wholesale; mention if data contain personal information.
- Never try to bypass the guard (comments tricks, multiple statements). If a legitimate need is blocked, say so and use the dedicated tool.
- Results reflect the data at query time; for large/volatile tables say so.

## Typical errors

| Error | Meaning / fix |
|---|---|
| `Only SELECT or WITH statements are allowed` | not a query, or starts with something else |
| `Keyword not allowed (read-only server): X` | a write/DDL word appears (even inside the statement) |
| `With ALLOWED_LIBS active every table/function must be qualified` | write `LIB.TABLE` |
| `Library not allowed (ALLOWED_LIBS): X` | outside the allowed list |
| `SQL0204` (not found) | wrong library/file name or no authority; check with `as400_table_list` |
| `SQL0206` (column not in table) | check field names with `as400_file_desc` |
| `Timeout (60000 ms)` | add filters; avoid whole-catalog scans; aggregate on keyed fields |
