---
name: as400-source-analysis
description: Analyze and explain a single IBM i program or file definition with as400_source_refs (files, called programs, /COPY, data areas, SQL tables, CL commands), as400_dds_describe (PF, LF, DSPF, PRTF layouts from DDS) and as400_source_read. Includes how to read fixed-format RPG and CL, how to structure a program explanation, and how to interpret the heuristic limits of the static analysis.
---

# Analyzing a program or a DDS definition

Prerequisites: `as400-mcp-basics`, and `as400-source-discovery` for locating the member.

## Method for "explain this program"

1. **Locate** the member (`as400_source_list` with a filter) and note its type and last-change date.
2. **Static references first**: `as400_source_refs` (library, file, member; optional `type` such as `RPGLE`, `CLLE`, `PF`;
   default is read from the member). It returns, according to the source type:
   - files used, with access mode (input, update, output, keyed);
   - programs and procedures called;
   - `/COPY` / `/INCLUDE` members;
   - data areas;
   - SQL tables referenced in embedded SQL;
   - CL commands (CL sources);
   - record formats and keys (DDS sources).
   This is **heuristic**: it does **not** see dynamic calls (`CALL` with a variable program name, `OVRDBF` redirections,
   `EXTFILE(variable)`, `QCMDEXC`). Treat it as a lower bound.
3. **Read the head** of the source (`as400_source_read`, lines 1–150): header comments, change log, `F` specs (files),
   `D` specs (data structures, prototypes, `EXTNAME`, `PREFIX`), entry parameters (`*ENTRY PLIST`, `PI`).
4. **Read the logic**: follow `*INZSR`, the main loop, then the subroutines/procedures named in step 2. Read in windows of
   ~300 lines; do not dump huge members whole.
5. **Resolve every external name**: for each file in `as400_source_refs` use `as400_file_desc`; for each called program
   use `as400_object_list`/`as400_source_list` and, if relevant, `as400_source_refs` again (call graph, depth 1–2).
6. **Explain** with sections: purpose, inputs/parameters, files read/written (and keys), called programs, main flow,
   business rules found in the code, anomalies (dead code, hard-coded values, unreachable branches, missing error
   handling), and what you could not determine.

## Reading fixed-format RPG (RPG III/IV, columns matter)

| Column 6 | Spec | Meaning |
|---|---|---|
| `H` | control | compile options |
| `F` | file | `FNAME IF/UF/O A E K DISK` (I=input, U=update, O=output, A=add, E=externally described, K=keyed); `EXTFILE(x)` = file name decided at run time; `USROPN` = opened by the program; `RENAME(a:b)` renames a record format |
| `D` | definition | `DS` data structure, `S` standalone, `C` constant, `PR`/`PI` prototype/interface; `EXTNAME(F)` = layout from file F; `PREFIX(P_)` |
| `I` | input | field/DS definitions (RPG III style) |
| `C` | calculation | opcode in columns 26–35 (`CHAIN`, `SETLL`, `READE`, `READ`, `WRITE`, `UPDATE`, `DELETE`, `EXFMT`, `CALL`, `CALLB`, `EVAL`, `IF`, `DOW`, `EXSR`, `KLIST`, `PARM`); resulting indicators in the right-hand columns (e.g. `70` = not found/EOF) |
| `O` | output | RPG III output specs |
| `P` | procedure | begin/end of a subprocedure |
| `*` in col 7 | comment | including the change log: "Recompiled for change to X" is history, not usage |

Free-format RPGLE (`/FREE` … `/END-FREE`, or `**FREE`) uses `dcl-s`, `dcl-ds`, `dcl-f`, `dcl-pr`, `monitor`, `exec sql`.
Useful patterns: `SETLL` + `READE` = read all records with a key; `CHAIN` = read one by key; `KLIST`/`%KDS` = key
definition (look at the `KFLD`s to know the key fields); `EXFMT` = display file I/O (interactive screen); `WRITE` to a
printer file = report; `EXEC SQL` = embedded SQL (cursor, SELECT INTO).

Reading CL: `PGM PARM(...)`, `DCL VAR`, `CALL PGM(...)`, `OVRDBF FILE(a) TOFILE(b)` (redirects a file used by the called
program: the program reads `a` but the data comes from `b`), `OVRPRTF`, `CHGDTAARA`/`RTVDTAARA`, `SBMJOB`, `MONMSG`,
`IF/ELSE/GOTO`. An `OVRDBF` before a `CALL` is the single most important thing to notice when tracing a data flow.

## `as400_dds_describe`: layouts of files, screens and reports

- Two ways: `library` + `object` (the compiled file; the tool finds the source and **warns if the source was modified
  after the object was compiled**), or `library` + `file` + `member` (a DDS source member).
- Covers PF, LF, DSPF, PRTF: record formats, fields (type, length, decimals, usage, line/position, text, heading),
  constants on screen/report, indicators, keys, select/omit, keywords (`DSPSIZ`, `CA03`, `CF05`, `OVERLAY`, ...).
- For PF/LF the **SQL catalog (`as400_file_desc`) is more reliable** than DDS (the source may be stale); use DDS for DSPF and
  PRTF, where it is the only way to see screen/report layout, and for LF select/omit logic.
- To understand an interactive program: DSPF record formats + function keys (`CAxx`/`CFxx`) + the `EXFMT` and indicators
  in the RPG. To understand a report: PRTF fields/positions, then compare with a real spooled file (`as400-spool-files`).

## Call graph and impact analysis

- *Who calls program X?* → `as400_source_search` with `text: "X"` limited to program source files, then keep lines that are
  `CALL`, `CALLB`, `CALLP`, prototype `PR`, or CL `CALL PGM(X)`; discard comment lines.
- *What does X call?* → `as400_source_refs` on X, plus a search for `CALL` with variable names for dynamic calls.
- Pattern of ERP naming (typical, verify per system): a family prefix per module, then a number; suffix letters often mean
  role (`C` control, `W` work/screen, `D` detail/update, `R` routine, `UI` user interface wrapper, `CL` CL wrapper). Treat
  such roles as **inferred** until you read the code.
- Compiled-object view: `as400_object_describe` on the `*PGM` gives bound modules, service programs and the source of origin.

## Output template

```
Program: LIB/FILE(MEMBER)  type, last change
Purpose: ...
Parameters: ...
Files: name — access — key — purpose
Calls: program — when/why
Flow: numbered steps
Business rules: ... (cite line numbers)
Anomalies / risks: ...
Verified vs inferred vs not examined: ...
```
