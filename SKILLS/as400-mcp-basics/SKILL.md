---
name: as400-mcp-basics
description: Foundations for working with an IBM i (AS/400) system through the my-mcp-as400 MCP server (tools named as400_*). Use FIRST whenever the task involves IBM i, AS/400, iSeries, RPG, CL, DDS, libraries, physical/logical files, IFS or spooled files and the as400_* tools are available. Explains the read-only scope, language/parameter-name switch (LOCALE), size limits, naming rules, how to choose the right tool, and how to report findings honestly.
---

# IBM i through the my-mcp-as400 MCP server: basics

## What the server is

A **read-only** MCP server (stdio) connected to one IBM i system with a dedicated profile. It exposes 27 tools
(`as400_*`), 5 resources and 7 prompts. It can read sources, database catalog and data, compiled-object metadata, the IFS
and spooled files. It **cannot** write, call programs, run CL commands or change anything. Never promise or attempt a
modification; if the user wants one, say it is outside this server and propose the change as text (a CL/SQL/RPG snippet
for them to run themselves).

## First steps in any session

1. Call `as400_system_info`: it confirms the connection, the IBM i version and the **active limits**
   (`SQL_MAX_ROWS`, `SOURCE_MAX_LINES`, `ALLOWED_LIBS`, ...). If it fails, stop and report the error; do not guess.
2. If the user did not name a library, call `as400_schema_list` with a `filter` (never list everything on a big system).
3. Pick the specialised skill for the job (table at the end).

## Language switch: parameter names depend on `LOCALE`

The server speaks English by default. With `LOCALE=IT` the **parameter names, enumerated values and result keys** change.
Always read the tool schema you were given: it shows which language is active. Mapping (English → Italian):

| EN | IT | EN | IT |
|---|---|---|---|
| `library` | `libreria` | `fromPage` | `daPagina` |
| `member` | `membro` | `toPage` | `aPagina` |
| `filter` | `filtro` | `clean` | `pulisci` |
| `path` | `percorso` | `status` | `stato` |
| `lineNumbers` | `numeriRiga` | `userData` | `datiUtente` |
| `fromLine` / `toLine` | `daRiga` / `aRiga` | `from` / `to` | `dal` / `al` |
| `text` | `testo` | `object` | `oggetto` |
| `type` | `tipo` | `attribute` | `attributo` |
| `limit` | `limite` | `depth` | `profondita` |
| `maxMembers` | `maxMembri` | `maxEntries` | `maxVoci` |
| `caseSensitive` | `maiuscole` | `length` | `lunghezza` |
| `parameters` | `parametri` | `maxFiles` / `maxFileBytes` | `maxFile` / `maxByteFile` |
| `field` | `campo` | `question` | `domanda` |
| `user` | `utente` | `name` / `date` | `nome` / `data` |
| `number` | `numero` | | |

Unchanged in both languages: `file`, `outq`, `job`, `query`, `offset`, `utf8`. Enumerated values:
`as400_table_list.type` = `all|physical|logical|views|sources` (IT: `tutti|fisici|logici|viste|sorgenti`);
`as400_routine_list.type` = `procedures|functions|all` (IT: `procedure|funzioni|tutte`);
`as400_ifs_list.type` = `all|file|directory` (IT: `tutti|file|directory`). Tool names never change.
Result keys follow the language too (`library/libreria`, `list/elenco`, `note/nota`, `total_lines/righe_totali`,
`dds_key/chiave_dds`, `found/trovate`). Rows returned by `as400_sql_query` are the user's table rows and are never translated.
Messages produced by IBM i itself (SQL errors, `CPF…`) are in the server job's language whatever `LOCALE` is.

## Rules that apply to every tool

- **Object names are at most 10 characters**, `[A-Z0-9_$#@]`, no spaces or dots. Long SQL names are not accepted by
  the object tools (library, file, member, object, outq). Use upper case.
- **Filters are SQL `LIKE`**: `%` = any string, `_` = one character. A filter without wildcards is an exact match.
  `file`, `member`, `filter` behave this way; `text` filters are substring matches; `as400_source_search.text` is a
  plain substring, **not a regex**.
- **Numbers come back as strings** in results (`"15"`): convert before comparing or summing.
- **Output is capped** and the result says so in a `note`/`nota` field (rows, lines, characters, members). A capped answer
  is *incomplete*: narrow the request or page through it, and tell the user it was truncated. Very large results may be
  saved to a file by the client instead of shown inline: read that file in parts, do not assume it was empty.
- **"Not found" can mean "no authority"**. The profile only sees what it is authorized to; an empty library may be a
  permission issue, not a missing object. Say so instead of asserting the object does not exist.
- **Libraries may be restricted** (`ALLOWED_LIBS`, `IFS_ALLOWED_PREFIXES`). The error names the rule; do not try to bypass it.
- **Always filter** on big systems. Production libraries have thousands of objects and thousands of members; unfiltered
  catalog queries time out (60 s default). Prefer a name prefix (`filter: "AB%"`), a description `text`, or a type.
- **Timeouts abandon the wait, not the work on the server.** If a call times out, narrow it; do not just retry.
- **Cost model:** catalog tools are fast; reading one source member is ~0.5 s; `as400_source_search` costs ~0.5 s *per
  member scanned* (default cap 150 members, 180 s limit); `as400_ifs_search` is bounded by file count/depth.

## Choosing a tool

| I want to... | Use |
|---|---|
| check the connection / limits | `as400_system_info` |
| find which libraries exist | `as400_schema_list` |
| list source files or members | `as400_source_list` |
| read a program's source | `as400_source_read` |
| find where text/field/file/program name appears in sources | `as400_source_search` |
| see what files/programs/data areas a program touches | `as400_source_refs` |
| see screen/printer/file layout from DDS | `as400_dds_describe` |
| list files/tables/views of a library | `as400_table_list` |
| find a field by name or description | `as400_column_search` |
| know fields/keys of a file | `as400_file_desc` (authoritative for PF/LF) |
| know logicals/indexes on a file | `as400_index_list` |
| know views/FKs/triggers depending on a file | `as400_where_used` |
| read a view's SQL | `as400_view_def` |
| read primary/unique/check/foreign keys | `as400_constraints` |
| list SQL procedures/functions | `as400_routine_list` |
| look at real data | `as400_sql_query` |
| list compiled programs/commands/data areas | `as400_object_list` |
| know a program's modules, service programs, source of origin, a data area value | `as400_object_describe` |
| browse/read files in the IFS | `as400_ifs_list`, `as400_ifs_stat`, `as400_ifs_read`, `as400_ifs_search`, `as400_ifs_read_binary` |
| find or read printed output, joblogs | `as400_spool_search`, `as400_spool_list`, `as400_spool_info`, `as400_spool_read` |

Which **program** uses a file is *not* answered by `as400_where_used` (it covers views/logicals/FKs/triggers only). For
programs, use `as400_source_search` on the file name, and confirm with `as400_source_refs` on the candidate members.

## Reporting honestly (always)

Separate three levels in any report or answer, and say which one each claim belongs to:

1. **Verified**: you read the code/data/catalog row that proves it (cite library/file/member/line or the query).
2. **Inferred**: deduced from names, conventions or descriptions (say "probably" and why).
3. **Not examined**: parts you did not scan (state the filters, caps and members you skipped).

Do not claim exhaustive coverage of a library you scanned only by description or by a capped search. Mention truncation notes.
Answer in the language the user writes in, whatever the server `LOCALE` is.

## Specialised skills

`as400-source-discovery`, `as400-source-analysis`, `as400-database-catalog`, `as400-sql-data-query`,
`as400-object-inspection`, `as400-ifs-files`, `as400-spool-files`, `as400-feature-reverse-engineering`.
