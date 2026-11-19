# Skills for the my-mcp-as400 server

Each folder is a self-contained Claude skill (a `SKILL.md` with `name` and `description` frontmatter). They teach a model
**how to use the 27 tools of the `my-mcp-as400` MCP server well**: which tool to pick, in which order, with which filters,
what the limits are and how to report findings honestly.

The skills describe the tools; they do not replace the server. The MCP server must be connected for any tool call to work
(see the project [README](../README.md)).

| Skill | Covers | Tools |
|---|---|---|
| [`as400-mcp-basics`](as400-mcp-basics/SKILL.md) | Start here: connection check, language (`LOCALE`), limits, naming rules, choosing a tool, reporting | `as400_system_info` and all others (routing table) |
| [`as400-source-discovery`](as400-source-discovery/SKILL.md) | Find and read RPG/CL/DDS/SQL sources, search text across a library | `as400_source_list`, `as400_source_read`, `as400_source_search` |
| [`as400-source-analysis`](as400-source-analysis/SKILL.md) | Understand one program: static references, call graph, DDS layouts, fixed-format RPG | `as400_source_refs`, `as400_dds_describe`, `as400_source_read` |
| [`as400-database-catalog`](as400-database-catalog/SKILL.md) | Explore the data model: libraries, files, fields, keys, logicals, constraints, views, routines | `as400_schema_list`, `as400_table_list`, `as400_column_search`, `as400_file_desc`, `as400_index_list`, `as400_where_used`, `as400_view_def`, `as400_constraints`, `as400_routine_list` |
| [`as400-sql-data-query`](as400-sql-data-query/SKILL.md) | Answer questions with read-only SELECTs on real data | `as400_sql_query` |
| [`as400-object-inspection`](as400-object-inspection/SKILL.md) | Compiled objects: programs, service programs, commands, data areas, data queues | `as400_object_list`, `as400_object_describe` |
| [`as400-ifs-files`](as400-ifs-files/SKILL.md) | Browse, read, hexdump and search the IFS | `as400_ifs_list`, `as400_ifs_stat`, `as400_ifs_read`, `as400_ifs_read_binary`, `as400_ifs_search` |
| [`as400-spool-files`](as400-spool-files/SKILL.md) | Find and read spooled files, joblogs and reports | `as400_spool_list`, `as400_spool_search`, `as400_spool_info`, `as400_spool_read` |
| [`as400-feature-reverse-engineering`](as400-feature-reverse-engineering/SKILL.md) | Workflow: map a business feature (e.g. "promotions") across an ERP's sources, data and modules, and write the report | all of the above |

## Installing

- **Claude Code:** copy (or symlink) the folders you want into `~/.claude/skills/` (all projects) or `.claude/skills/`
  (one project). Example: `cp -r SKILLS/as400-* ~/.claude/skills/`.
- **Other Claude environments (Claude.ai, API, Agent SDK):** upload or reference the skill folders the way that product
  expects skills; each folder is independent and only needs its `SKILL.md`.
- Load `as400-mcp-basics` together with whichever specialised skill you use: the specialised ones assume its conventions.

## Language and parameter names

The skills use the **English** parameter names (the server default, `LOCALE=EN`). If the server runs with `LOCALE=IT`,
parameter names, enumerated values and result keys are Italian: the mapping table is in `as400-mcp-basics`.
