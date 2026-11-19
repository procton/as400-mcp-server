---
name: as400-database-catalog
description: Explore the Db2 for i data model with the as400_schema_list, as400_table_list, as400_column_search, as400_file_desc, as400_index_list, as400_where_used, as400_view_def, as400_constraints and as400_routine_list tools. Use to find libraries, physical/logical files, tables and views, locate a field, read keys, logicals, constraints, view definitions and SQL routines, and to build an entity/relationship picture of an application such as an ERP.
---

# Database catalog exploration

Prerequisite: `as400-mcp-basics`. To read the *data* itself, use `as400-sql-data-query`.

## IBM i vocabulary

- **Library** = schema. **PF** = physical file (table), **LF** = logical file (keyed/filtered view or index over a PF),
  **member** = the data partition of a file (usually one). **Record format** = row layout; **DDS** = the source that
  defines PF/LF. A file may also be an **SQL table/view/index** created with DDL.
- Fields have a **short name** (≤ 10 chars, used by RPG) and optionally a **long SQL name**, plus a text/heading. ERP
  field names are usually prefixed by the file's mnemonic (e.g. `MGLPC00F` → fields starting `LPC…`).
- Naming convention in many ERPs: `…F` = physical file, `…W`/`…WW` = work copy, `…L01`/`…L02` = logicals. Treat as inferred.

## Tools

| Tool | Parameters | Returns / use |
|---|---|---|
| `as400_schema_list` | `filter` (LIKE), `limit` | libraries with description and owner; honors `ALLOWED_LIBS` |
| `as400_table_list` | `library`; `type` (`all`/`physical`/`logical`/`views`/`sources`); `filter`; `text`; `limit` | files with description, field count, record length |
| `as400_column_search` | `library`; at least one of `field` (LIKE on short/long name) or `text` (substring in description/heading); `file` (LIKE); `limit` | where a field lives |
| `as400_file_desc` | `library`, `file` | type, description, fields (name/type/length/decimals/text), keys |
| `as400_index_list` | `library`, `file` | DDS key of the file plus the logicals/indexes built on it (keys, uniqueness, record format) and non-indexed LFs |
| `as400_where_used` | `library`, `file` | views, logicals/indexes, child tables through foreign keys, triggers that depend on the file. **Does not detect RPG/CL programs.** |
| `as400_view_def` | `library`, `file` (the system name of the view) | SQL text of the view |
| `as400_constraints` | `library`, `file` | primary, unique, check and foreign keys with referenced table and rules. Legacy DDS files usually have none: relationships are implicit |
| `as400_routine_list` | `library`; `type` (`procedures`/`functions`/`all`); `filter`; `limit` | SQL and external procedures/functions |

## Recipes

**Discover the files of a functional area.** `as400_table_list` with `filter` on the prefix (`"MGLP%"`) and/or `text` on a
word in the description (`"promoz"`), `type: physical`. Do a second call with `type: logical` to see access paths.
Never call it without a filter on a production library.

**Find a field.** `as400_column_search` with `field: "%CODCLI%"` (name) or `text: "customer"` (description), optionally
restricted by `file`. At least one of `field` or `text` is required. Results show every file that contains the field:
common fields across files = candidate join keys.

**Understand one file.** `as400_file_desc` (fields, keys) → `as400_index_list` (access paths) → `as400_constraints` (declared
relations) → `as400_where_used` (dependents) → a few sample rows with `as400_sql_query` (`FETCH FIRST 5 ROWS ONLY`).

**Infer relationships in a legacy model** (no foreign keys): join on key fields that share name stem, type and length across
files (`as400_column_search`), then **check with data** (a join that returns rows, orphans = `LEFT JOIN … WHERE x IS NULL`)
before asserting it. State relationships found this way as *inferred, validated on sample data*.

**Which programs use this file?** Not here. Use `as400_source_search` for the file name in the program source files, and
read the matching lines (see `as400-source-discovery`). Change-log comments ("recompiled for change to FILE") also appear.

**Work files and copies.** ERPs often keep a master file plus work/temporary copies and per-document variants (`…W`, `…WW`,
`…OF`). Identify them by identical field lists (`as400_file_desc` on both) and by `EXTFILE`/`OVRDBF` in programs.

## Pitfalls

- `as400_table_list` and the catalog views are slow without filters on large libraries (thousands of files): use prefix
  filters. A `GROUP BY` over the whole catalog of a big library can time out.
- `file_desc` takes the **file name (≤ 10 chars)**; for SQL objects with long names use the system name.
- For source files (`type: sources`) `as400_file_desc` redirects to `as400_source_list`.
- Fields can be **packed/zoned decimals** with implied decimals (see `decimals`); dates are often numeric `YYYYMMDD` or
  `CYYMMDD`; flags are often `'S'`/`'N'`, `'1'`/`'0'` or `' '`. Verify with `SELECT DISTINCT` before documenting meanings.
- CCSID/text conversion: text fields are returned trimmed; accented characters depend on the file's CCSID.
- A library the profile cannot see appears empty: do not conclude "no files" without considering authority.

## Output

When documenting a data model, give per file: name, description, key, main fields (meaning verified or inferred), related
files (with the join fields), access paths, and approximate size if you counted rows (`SELECT COUNT(*)`, which is cheap on
one file).
