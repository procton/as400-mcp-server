---
name: as400-object-inspection
description: Inspect compiled IBM i objects (programs *PGM, service programs *SRVPGM, commands *CMD, data areas *DTAARA, data queues *DTAQ, files, modules) with as400_object_list and as400_object_describe. Use to inventory a library by object type, find which source an object was compiled from and when, see bound modules and service programs, read the value of a data area, and detect programs without source or with stale source.
---

# Object inspection

Prerequisite: `as400-mcp-basics`.

## Tools

### `as400_object_list`
`library` required. Optional: `type` (types separated by spaces, e.g. `"*PGM *SRVPGM"`, default `*ALL`), `filter` (name,
LIKE), `text` (substring in the description), `attribute` (e.g. `RPGLE`, `CLLE`, `PF`, `LF`, `DSPF`, `PRTF`), `limit` (default 200).

Returns object, type, attribute/language, description, creation and change dates, **source file/library/member of origin**
and source timestamp. A name prefix filter (`"OC%"`) is passed to the system API as a generic name and is fast; **unfiltered
listings of libraries with thousands of objects can time out**: always filter.

Types worth knowing: `*PGM` (program), `*SRVPGM` (service program), `*MODULE`, `*CMD` (command), `*DTAARA` (data area),
`*DTAQ` (data queue), `*FILE` (PF/LF/DSPF/PRTF/SAVF), `*MENU`, `*MSGF`, `*OUTQ`, `*JOBD`, `*USRSPC`, `*BNDDIR`, `*DTAARA`.

### `as400_object_describe`
`library`, `object`, `type` required (type with the asterisk: `*PGM`, `*SRVPGM`, `*DTAARA`, `*FILE`, ...).
- Always: dates, owner, source of origin.
- `*PGM`/`*SRVPGM`: bound modules and service programs (what it was linked with), activation group, program info.
- `*DTAARA`: **the current value** and its length/type (often configuration or a sequence counter).
- Sections the system cannot provide are reported with an `error` field while the rest is still returned.

## Recipes

**Inventory a library.** `as400_object_list` per type and prefix; count by attribute; list `*CMD` and `*DTAARA` to find
user entry points and configuration.

**From a program name to its source.** `as400_object_describe` → source file/library/member → `as400_source_read`.
If the source member is missing, or the member's change date is **later than the object's compile date**, the source may not
match the running program; say so and treat conclusions as unverified. `as400_dds_describe` with `object` makes the same
comparison for files.

**Programs without source / sources without object.** Compare `as400_object_list` (`*PGM` with attribute `RPGLE`) with
`as400_source_list` members for the same names. Differences are candidates for dead or externally built objects.

**Find configuration.** `as400_object_list` with `type: "*DTAARA"` then `as400_object_describe` on each to read the value
(e.g. company code, next number, flags). Programs read them via `RTVDTAARA`/`IN`; search the sources for the data area name.

**Find commands that start a feature.** `as400_object_list` `type: "*CMD"` with a description `text`; the command's
`CPP` (command processing program) is then found with `as400_object_describe`/source search.

**Understand deployment.** Several libraries may hold the same program at different versions (test/production). Compare
compile dates and source timestamps rather than assuming.

## Pitfalls

- Objects the profile cannot see do not appear: a short list may mean missing authority.
- `attribute` for service programs/commands/menus may be empty; filter by `type` instead.
- A program created from a CL or RPG source in a different library (copy library, QRPGLE870, ...) shows that library in the
  origin: read the source from **there**.
- `as400_object_list` describes objects; for the *contents* of a file use `as400-sql-data-query`, for its structure
  `as400-database-catalog`.
