---
name: as400-feature-reverse-engineering
description: End-to-end workflow to map a business feature (promotions, pricing, orders, invoicing, stock, lots, web orders, contracts...) across an IBM i ERP or legacy application using the as400_* MCP tools, and to produce a structured report of the sources involved, the data model, the call chains and the links with the ERP modules. Use when asked to "analyze all the sources and find the X management", document a module, or assess the impact of a change.
---

# Reverse-engineering a feature across an IBM i ERP

Prerequisites: `as400-mcp-basics`; the specialised skills `as400-source-discovery`, `as400-source-analysis`,
`as400-database-catalog`, `as400-object-inspection` are used inside this workflow (and `as400-sql-data-query`, `as400-spool-files` for evidence).

The goal is a report that is **useful and honest**: what the feature is made of, how the pieces connect, and exactly how much
of the library you really examined. The library can contain tens of thousands of members; a literal "read everything" is
not possible with these tools (about 0.5 s per member scanned), so work by **layers of evidence**.

## Phase 0 — Scope and vocabulary

- Confirm library/libraries and source files (`as400_source_list` without `file`).
- List the **synonyms** of the feature in the system's language and the business: for promotions: `promoz`, `promo`,
  `offerta`, `sconto`, `coupon`, `premio`, `listino promozionale`, `campagna`, `punti`, `pacchetto`. Ask the user if the
  term may have a different meaning in this ERP.
- Learn naming conventions from a quick sample: module prefixes (2–3 letters), role suffixes, work-file patterns.
  Write them down as *hypotheses* to be confirmed.

## Phase 1 — Cheap pass: names and descriptions (verified-by-metadata)

1. `as400_source_list` for each program source file with `filter` = candidate prefixes; read the **descriptions**.
2. `as400_table_list` with `text` = synonyms (`type: physical`, then `logical`), and `as400_object_list` with `text` =
   synonyms for `*PGM`, `*CMD`, `*DTAARA`, `*FILE` (DSPF/PRTF).
3. `as400_column_search` with `text` = synonyms to find fields that carry the concept (and which files contain them).
4. Result: a **candidate list** (members, files, objects). This is evidence by description only; mark it so.

## Phase 2 — Seed the data model

- Pick the master file(s) of the feature from phase 1 and run `as400_file_desc`, `as400_index_list`, `as400_constraints`,
  `as400_where_used`. Identify key fields, status fields, validity dates, and the **linked entities** (customer, supplier,
  item, price list, document).
- Sample a few rows (`as400-sql-data-query`) to understand codes and status values. Count rows to size the feature.
- Look for **families of files with the same layout** (master, work copy, per-document reference files, history/snapshot
  copies): `as400_table_list` by name stem, then compare `as400_file_desc`.

## Phase 3 — Content scans: who really uses the seed files (verified-by-content)

For each seed file name and each key token (a file name, a data area, a work-item number), run `as400_source_search`
restricted to program source files, **in batches by member prefix** so that no batch exceeds the member cap:
`member: "OC%"`, then `"OF%"`, `"GA%"`, `"FT%"`, ... (raise `maxMembers`/`limit` per batch; `limit` max 1000).

Interpret every hit line, do not just count them:
- `F…` specifications, `EXTNAME(...)`, `CHAIN/SETLL/READE/WRITE/UPDATE` = real use (say whether read or update).
- `OVRDBF`, `EXTFILE`, `EVAL name = 'FILE'` = file **redirection or dynamic name**: record both ends.
- `* Recompiled for change to FILE` = a dependency was *recompiled* because the file's layout changed: a real dependency
  (compile-time) but not a statement about logic; the work-item numbers in these comments give a **change history**.
- Comments describing the feature are good seeds for the next search.

Record per hit: module (prefix), member, role (maintenance, engine, document application, print, interface), access.
Keep a table of batches run with `members_scanned` and any cap note: it becomes the coverage statement.

## Phase 4 — Structure of the chain

- For the key programs run `as400_source_refs` (files, calls, data areas, copy members) and read the head and the main flow
  with `as400_source_read`. Always read CL wrappers (`…CL`): they hold overrides, flags, job submission and the **order of
  calls**.
- Build the chain: UI/menu → control program → engine/functions → data files → print/interface. Find the **entry points**:
  `*CMD` objects, menu options, `…UI` wrappers, data queues, web/API programs.
- Resolve variants: different document cycles (customer order, supplier order, quotation, invoice) usually reuse the same
  engine through different files (look for per-document file families and for flags that switch them).
- Find configuration: data areas (`as400_object_describe` for values), parameter files (small PF with a code/value layout).
- Find outputs: PRTF/report programs (`as400_dds_describe`), spooled examples (`as400_spool_search` by user data/name).

## Phase 5 — Links with modules

Make a matrix *feature component × ERP module* where each cell is one of: **verified** (a source line/query proves it),
**inferred** (naming/description/convention), or **absent in what was scanned**. Typical modules: master data (customers,
suppliers, items), price lists, sales orders/quotations, purchasing, invoicing/accounting, warehouse/stock/lots, contracts
and rebates, e-commerce/web, services (base, utility), tables/parameters, batch/job control.

## Phase 6 — The report

Use this structure (adapt the headings; language of the user):

1. **Summary** (what the feature is, in a paragraph, and the architecture in 5 lines).
2. **Components by function** (maintenance, applicability engine, document application, contracts/rebates, interface/web,
   reports/prints, batch/utilities): table of source members/objects with type, short purpose, evidence level.
3. **Data model**: master files, work copies, reference/history files, keys, relationships (validated on data), status codes.
4. **Flows**: step-by-step (including overrides and flags), from entry point to data written.
5. **Links with modules**: the matrix of phase 5.
6. **Evolution / change history** if change-log comments and work items exist.
7. **Naming conventions** observed.
8. **Coverage and limits**: what was verified by content, what only by description, which members/prefixes were *not*
   scanned, the caps and timeouts encountered, objects without source, possible dynamic calls, anything stale.
9. **Suggested next steps** (the unscanned areas in order of value).

Every statement carries its evidence level. Do not present inferred roles (from names) as facts. If the user asks for
"all sources", explain the scan model and give the real coverage instead of implying completeness.

## Practical tips

- Big outputs may be saved to a file by the client; read them in parts. Summarise intermediate results in your own notes
  between phases so that the context stays small.
- Prefer many narrow searches to one wide one; stop a scan family when it produces no new member for the feature.
- A work-item or ticket number found in change logs is a great pivot: search for it to find all sources touched by the same
  change, and to date the feature's evolution.
- Cross-check with data: a file that appears in many sources but has zero rows (`COUNT(*)`) is probably unused here.
- Save the final report as Markdown when the user asks for a file; this server cannot write files, so write it with the
  client's own file tools.
