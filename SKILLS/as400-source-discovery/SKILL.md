---
name: as400-source-discovery
description: Find and read IBM i source code (RPG, RPGLE, CL, CLLE, DDS, SQL, copy members) with the as400_source_list, as400_source_read and as400_source_search tools. Use to list source files and members of a library, read a program's source, or search text (a file name, field, program name, keyword, work-item number) across many members, while respecting the cost and limits of the scan.
---

# Source discovery: list, read, search

Prerequisite: `as400-mcp-basics` (naming, limits, reporting). Parameter names below are the English ones.

## Tools

### `as400_source_list` — what sources exist
- `library` (required). `file` optional, `filter` optional (LIKE on name).
- Without `file`: lists the **source files** of the library (QRPGLESRC, QCLSRC, QDDSSRC, QRPGLE… including custom ones).
- With `file`: lists the **members** with type, description, line count and dates.
- On libraries with thousands of members always pass `filter` (e.g. `"OC%"`). A member listing of a whole big source file
  can exceed the output cap or time out; a name-prefix filter is fast.
- The member **description** (text) is valuable: ERP vendors often write a meaningful description. The tool returns it, so
  use it as the first, cheap discovery pass before any content scan.

### `as400_source_read` — the text of one member
- `library`, `file`, `member` required. Optional `fromLine`/`toLine` (1-based, inclusive), `lineNumbers` (default true).
- Preserves leading blanks, which matters in **fixed-format RPG** (column 6 = specification type). Do not re-indent.
- Long members: read in windows (for example 1–400, then 401–800). `SOURCE_MAX_LINES` and `SOURCE_MAX_CHARS` cap one answer;
  the result reports `total_lines` so you know how much is left. Start with the first 150 lines (header, change log,
  file declarations, prototypes) before reading the body.
- Member name must exist exactly; otherwise the error `Member or source file not found: LIB/FILE(MEMBER)`. Check with `as400_source_list`.
- Technically reads through a temporary alias in QTEMP; this is the only operation using a non read-only connection and it
  is limited to fixed statements. Nothing is written to the application libraries.

### `as400_source_search` — text across members
- `library`, `text` (≥ 2 chars, **substring, not regex**) required. Optional: `file` (LIKE, default all source files),
  `member` (LIKE), `caseSensitive` (default false), `limit` (matching lines, default 100, **max 1000**), `maxMembers`
  (default `SEARCH_MAX_MEMBERS`, 150).
- Returns `file`, `member`, `line`, `text` for each match, plus `members_scanned` and a note when a cap was reached.
- **Cost ≈ 0.5 s per member scanned.** A 180 s limit applies. If the library has more members than `maxMembers`, the
  scan is *partial*: narrow with `file` and `member` prefixes and run it in several batches (e.g. `member: "OC%"`, then
  `"OF%"`, ...), raising `maxMembers`/`limit` for each batch. Report which batches you ran.
- If you hit `limit`, the matches are truncated in scan order, not ranked: narrow the text or the member filter.
- Search for the **most selective token**: a unique file name (`MGLPC00F`), a field name, a called program name, a
  work-item number. Generic words (`PROMO`) produce noise.
- Case-insensitive by default, which is right for RPG (it is case-insensitive) and for comments.

## Efficient strategies

1. **Description first** (cheap): `as400_source_list` with `filter` on a name prefix, scanning the descriptions; or
   `as400_object_list`/`as400_table_list` with a description `text`. Names and descriptions locate 80 % of a feature.
2. **Content second** (costly): `as400_source_search` on the selective tokens found, restricted to the source files that
   hold programs (QRPGLESRC, QCLSRC, ...) and to prefixes of interest.
3. **Read last**: open only the members that the previous steps identified; read header/change log first.
4. Use change-log comments: ERP sources often carry lines such as "Recompiled for change to FILEX" or work-item
   numbers. A search for a **file name** therefore finds both the real users *and* programs merely recompiled for a
   change; read the line text to tell them apart (`F…` spec, `CHAIN`/`READE`, `EXTNAME` = real use; a `*` comment = history).
5. Look for the member types you expect: RPGLE/RPG (programs), CLLE/CLP (wrappers and batch jobs), SQLRPGLE, PF/LF/DSPF/
   PRTF (DDS), SQL (DDL), copy members (`/COPY`, `/INCLUDE`) and prototypes.

## Pitfalls

- A search finds text in **sources only**; if the source of a compiled object is missing or stale the search will not see it.
  `as400_object_describe` shows the object's source file/member and compile date for comparison.
- Dynamically built names (`EVAL W_FILE = 'OFMOV06F'`, `OVRDBF`, `CALL` with a variable) appear as string literals:
  search for the literal, then read the context.
- Source can differ across library versions (e.g. `QRPGLE870` vs `QRPGLESRC`): state which file you used.
- Do not read dozens of members "to be sure". Decide from `as400_source_refs` (see `as400-source-analysis`) or search hits first.
