---
name: as400-spool-files
description: Find and read IBM i spooled files (printed output, reports, joblogs, QPJOBLOG, QSYSPRT, DSPLY output, dumps, invoices, labels) with as400_spool_list, as400_spool_search, as400_spool_info and as400_spool_read. Use to locate a report by user, name, job, date or user data, read its pages as text, understand its attributes and layout, diagnose job failures from joblogs, and compare a real printout with a print-file definition.
---

# Spooled files and output queues

Prerequisite: `as400-mcp-basics`. English parameter names: `library`, `outq`, `file`, `number`, `user`, `job`, `filter`,
`userData`, `status`, `type`, `from`, `to`, `limit`, `fromPage`, `toPage`, `clean`.

## Concepts

A spooled file is identified by **name + number + job (`number/user/name`)** and sits in an **output queue** (`library/outq`).
Several spooled files can share a name: disambiguate with `number`, `user` or `job`. Data types: `*SCS` (classic text
stream), `*AFPDS` (page-layout), `*LINE`, `*USERASCII`. `USRDTA` (user data) is a 10-character label that applications
often set to identify the document.

## Tools

| Tool | Purpose | Notes |
|---|---|---|
| `as400_spool_list` (`library`, `outq?`, `user?`, `filter?`, `limit?`) | without `outq`: queues of a library with file counts and status; with `outq`: files, **most recent first** | default 100, capped by `SPOOL_MAX_FILES` |
| `as400_spool_search` (`user`, `file`, `job`, `userData`, `status`, `type`, `from`, `to`, `library`, `outq`, `limit`) | search across **all queues**; also returns the queue to use with `spool_read` | **needs at least one filter** among user, file, job, userData, from or library; dates `YYYY-MM-DD`; `status` e.g. READY, HELD, PRINTING, SAVED; `type` e.g. `*SCS`, `*AFPDS` |
| `as400_spool_info` (`library`, `outq`, `file`, + `number`/`user`/`job`) | full attributes: pages, copies, form type, page size, lpi/cpi, fonts, overlays, the program that opened it | use before reading a big file |
| `as400_spool_read` (same id + `fromPage`, `toPage`, `clean`) | content as text, pages separated by form feeds | `clean` removes leftover control characters (default true) |

## Workflow

1. **Find**: `as400_spool_search` with the narrowest known filters (`user`, `file: "QPJOBLOG"`, `from`/`to`, `userData`).
   If the system is busy add a date range; never scan "everything".
2. **Choose** the right entry: newest for the right job/user; the result lists `file`, `number`, `queue`, `user`, `job`,
   `pages`, `created`.
3. **Check size**: `as400_spool_info` (pages, type). For many pages read selectively.
4. **Read**: `as400_spool_read` with `library`, `outq`, `file`, and `number` (needed when the name repeats; the error lists the
   most recent numbers). Page windows: `fromPage: 1, toPage: 3`. The result states `pages_in_text`, `pages_returned`
   and truncation (`SPOOL_MAX_CHARS`, `SPOOL_MAX_READ_BYTES`).
5. **Interpret** and summarise; quote the relevant lines with page references.

## Joblogs and error diagnosis

- Joblogs are spooled files named `QPJOBLOG` (user data often the job name). Search by `user` and date, read the last
  pages first: errors cluster at the end.
- Look for message IDs (`CPF####`, `MCH####`, `CPD####`, `SQL####`, `RNX####`, `CEE####`), the **first** error (later ones are
  often consequences), the program/module and statement number, and the job's command that started it.
- State the likely cause and how to verify it from sources (`as400-source-discovery`: search the program name or statement).
  Do not claim the cause is certain from one joblog line.

## Reports and layouts

- Read a printout to learn what a report shows: headings, columns, totals, sort order, selection criteria on the first
  page. Combine with the print file's definition (`as400_dds_describe` on the PRTF, or the RPG/`O` specs) to map columns
  to database fields, then verify the numbers with `as400_sql_query`.
- **`*SCS`** text is rendered with correct line breaks and pages. **`*AFPDS`** returns only the text: positions, overlays
  and images are lost and columns may appear merged or glued; the tool adds a note. For the layout use the PRTF/DDS.
- Accented letters: the text arrives through a Host Print Transform (default charset `cp850`). If you see wrong characters
  (`Ô` for `È`, `?`, boxes) report it: the administrator changes `SPOOL_CHARSET` (`latin1`, `utf-8`) in the server `.env`.

## Limits

- Read-only: the tools cannot hold, release, delete, move or print spooled files.
- Reading needs authority on the output queue and the file: otherwise "not found" may mean "not authorised".
- Very large files are read up to a byte cap: later pages are reported as not included.
- Output queues in `QUSRSYS`/`QGPL` are the usual starting points; many shops use dedicated libraries/queues per application.
- Personal or financial data may be on printouts; summarise rather than reproduce when not needed.
