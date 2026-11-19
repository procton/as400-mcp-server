---
name: as400-ifs-files
description: Browse, read, hexdump and search the IBM i Integrated File System (IFS: /home, /tmp, /QOpenSys, /QIBM, stream files, logs, scripts, configs, exports) with as400_ifs_list, as400_ifs_stat, as400_ifs_read, as400_ifs_read_binary and as400_ifs_search. Use to locate configuration files, read logs and scripts, inspect binaries or PDFs, handle CCSID problems and search text in a directory tree.
---

# IFS: browse, read, search

Prerequisite: `as400-mcp-basics`. Parameter names are English (`path`, `depth`, `filter`, `type`, `maxEntries`, `fromLine`,
`toLine`, `lineNumbers`, `utf8`, `offset`, `length`, `text`, `caseSensitive`, `limit`, `maxFiles`, `maxFileBytes`).

## Path rules

- Paths must be **absolute** (`/home/PROFILE/file.txt`), with **no `..`** components.
- If `IFS_ALLOWED_PREFIXES` is active, only those prefixes are readable and **symbolic links are not followed or
  listed** (a link could point outside the allowed area). Use the real target path when it is inside the prefixes.
- `/QSYS.LIB` (libraries, sources, objects) is not an IFS area for these tools: use the source/object tools. `as400_ifs_search`
  does not work in special file systems (`/QSYS.LIB`, `/QDLS`, ...).

## Tools

| Tool | Use |
|---|---|
| `as400_ifs_list` (`path`, `depth` 1–6, `filter` LIKE, `type` `all/file/directory`, `maxEntries` default 500, max 2000) | directory content, optionally in depth |
| `as400_ifs_stat` (`path`) | type, size, owner, **CCSID**, modification date; entry count for directories |
| `as400_ifs_read` (`path`, `fromLine`, `toLine`, `lineNumbers`, `utf8`) | text file, decoded by the file's CCSID; the line window is chosen on the server so large files are fine |
| `as400_ifs_read_binary` (`path`, `offset`, `length` default 256) | hex + ASCII dump, up to `IFS_BINARY_MAX_BYTES` |
| `as400_ifs_search` (`path`, `text` ≥ 2 chars, `depth` default 3, `filter` file-name LIKE, `caseSensitive`, `limit` ≤ 1000, `maxFiles`, `maxFileBytes` default 1,000,000) | substring search in text files under a directory |

## Workflow

1. **Orient**: `as400_ifs_stat` on the starting path, then `as400_ifs_list` with `depth: 1`. Go deeper only where needed
   (each directory costs about a second; very populated directories are slower).
2. **Filter**: `filter: "%.sh"`, `"%.log"`, `"%.properties"`, `"%.csv"`; `type: file`.
3. **Check the type before reading**: `as400_ifs_stat` shows CCSID and size. Check `CCSID 65535` = binary/untagged.
4. **Read text** with a window: `fromLine: 1, toLine: 200` for a log head; for the tail of a big log you need its line count
   first (read a small window and use the reported total, then request the last lines).
5. **Search a tree** with `as400_ifs_search`: specific text, specific `filter` (`%.log`), small `depth`. It skips binary and
   very large files and stops at `IFS_SEARCH_MAX_FILES`: the result says when the scan was cut short; report it.
6. **Binary or unreadable**: `as400_ifs_read_binary` from `offset 0`, 256–1024 bytes: recognise signatures
   (`%PDF`, `PK` zip, `7F 45 4C 46` ELF, `1F 8B` gzip, SAVF) and report what the file is; do not try to "read" it as text.

## Encoding problems

- Text appears with wrong letters (accents mangled, `Ã¨`): the CCSID tag is wrong (e.g. a UTF-8 file tagged 37).
  Re-read with `utf8: true`; confirm the tag with `as400_ifs_stat`.
- `CCSID 65535`: untagged binary. `as400_ifs_read` will not return readable text: use `as400_ifs_read_binary`, or try
  `utf8: true` if you know it is text.
- ASCII vs EBCDIC: files created by PC tools are usually UTF-8/ASCII, files made by native programs EBCDIC (CCSID 37/280/500/1144).
  The tool decodes based on the tag, so a correct tag matters more than the content.

## What you typically find

- Home directories of profiles: scripts, `.profile`, exports, logs.
- Integration folders: CSV/XML/JSON exchange files, EDI, configuration `.properties`/`.ini`/`.xml`, certificates.
- `/QIBM/UserData/...`: product configuration (web servers, Java, HTTP).
- `/tmp`: temporary files; do not treat them as stable.

## Limits and safety

- Read-only: no create/modify/delete/rename; no execute.
- Configuration files may contain **credentials**; do not echo secrets in answers; mention that the file contains them.
- Large outputs are capped (`IFS_MAX_LINES`, `IFS_MAX_CHARS`); the result states truncation, then continue with
  `fromLine`/`toLine`.
- Not found = missing or **no authority**: the message says "directory empty, nonexistent or no authority".
