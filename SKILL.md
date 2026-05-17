---
name: es-search
description: 'Fast file search on Windows using Everything CLI (es.exe). Use this skill in TWO situations: (1) when the USER asks to find, locate, or list files by name, pattern, extension, path, size, date, or attributes — phrases like "find all *.cpp files", "where is config.json", "files larger than 2 GB", "files modified today", "duplicate files", "list all headers in src/"; (2) when CLAUDE ITSELF is about to search for files — i.e. before reaching for the Glob tool, Get-ChildItem -Recurse, or a PowerShell find command on Windows. This skill should be Claude''s default file-search strategy on Windows: faster than Glob on large drives, works across all indexed disks, supports rich filters (size, date, attributes, duplicates, media metadata). Do NOT use for content search inside files (use Grep instead) or for listing a single flat directory.'
license: MIT
---

# es-search — Fast File Search via Everything CLI

`es.exe` is the CLI for [Everything](https://www.voidtools.com/) — a Windows
search tool that indexes the NTFS MFT and returns results instantly, regardless
of drive size or file count.

It accepts the full **Everything search syntax**
(https://www.voidtools.com/support/everything/searching/) plus CLI-specific
switches (https://www.voidtools.com/support/everything/command_line_interface/).

## Availability check

Before using `es.exe`, verify it is reachable:

```powershell
where.exe es 2>$null
```

If the command is not found **or** returns exit code 8 (Everything service not
running), fall back to the standard tools (see **Fallback** section below).

Quick availability check pattern:

```powershell
$esOutput = es -version 2>&1
if ($LASTEXITCODE -eq 8) {
    # Everything service not running
} elseif ($LASTEXITCODE -ne 0 -and -not ($esOutput -match 'ES ')) {
    # es.exe not installed
}
```

Note: `es` uses **single-dash** long flags (`-version`, `-help`, `-regex`).
GNU-style `--version` returns "Error 6: Unknown switch" — but the help text
is still printed to stderr, which can mislead automated checks. Match on the
`ES ` prefix in stdout, not on exit code alone.

## CRITICAL: argv tokenization rule

`es.exe` parses **each argv element as a single search token**. Whitespace
*inside* a single quoted argument is treated as a **literal substring** of the
filename, NOT as a token separator. This trips up almost every multi-operator
example you'd write naively.

```powershell
# WRONG — one argv with spaces; es looks for literal substring "*.cpp | *.h"
es -get-result-count "*.cpp | *.h"          # → 0
es -get-result-count "*.log !temp"          # → 0
es -get-result-count "<*.cpp | *.h> !test"  # → 0

# RIGHT (option A) — each operator/operand is its own argv
es -get-result-count "*.cpp" "|" "*.h"            # → 104 631
es -get-result-count "*.log" "!temp"              # → 4 270
es -get-result-count "<*.cpp" "|" "*.h>" "!test"  # works

# RIGHT (option B) — single argv, NO spaces around operators
es -get-result-count "*.cpp|*.h"            # → 104 631
es -get-result-count "*.log!temp"           # AND-NOT, no space
es -get-result-count "<*.cpp|*.h>!test"     # grouping, no spaces
```

Empirical rule: if you need spaces between operators (for readability) — split
into separate quoted argv elements. Otherwise glue everything inside one
quoted string. This is **not a PowerShell quirk** — `cmd /c 'es "*.cpp | *.h"'`
behaves identically.

Prefix-functions (`ext:`, `size:`, `dm:`, `dupe:`, `path:`, `regex:`) always
take ONE argv: glue the argument with `:` or wrap in `<...>`.

## Quick reference

| What | Syntax | Argv form |
|------|--------|-----------|
| Wildcard | `*.log`, `config*.json`, `?ame.txt` | single |
| AND (default) | `report` `2024` (two argv) or `*report*2024*` | split / glued |
| OR | `*.cpp` `\|` `*.h` (three argv) or `*.cpp\|*.h` | split / glued |
| NOT | `*.log` `!temp` (two argv) or `*.log!temp` | split / glued |
| Grouping | `<*.cpp\|*.h>!test` (one argv, no spaces) | glued |
| Exact phrase | `"my file.txt"` (escape inner quotes from PS) | single |
| Extension list | `ext:cpp;h;hpp` | single |
| Files only | `/a-d` | single |
| Folders only | `/ad` | single |
| Path scope | `-path "C:\dir"` | flag + value |
| Match against full path | `-p` or `path:foo` | flag / prefix |
| Regex | `-r "pattern"` (alone) or `regex:pattern` | flag / prefix |
| Size | `size:>2gb`, `size:1gb..5gb`, `size:empty` | single |
| Date modified | `dm:today`, `dm:2024`, `dm:2024-01-01..2024-12-31` | single |
| Date created / accessed | `dc:thisweek`, `da:yesterday` | single |
| Attributes | `attrib:H` (hidden), `attrib:R` (readonly) | single |
| Zero-byte files | `size:0` (NOT `empty:` — see below) | single |
| Empty folders | `empty:` | single |
| Duplicates by name | `dupe:<expr>` | single |
| Duplicates by size | `sizedupe:<expr>` | single |
| Limit results | `-n 50` | flag |
| Offset / pagination | `-o 100 -n 50` | flag |
| Count only | `-get-result-count` | flag |
| Sum file sizes (bytes) | `-get-total-size` | flag |
| Sort | `-sort size-descending` | flag |
| Structured output | `-csv` / `-tsv` / `-efu` (no JSON) | flag |

## Core search patterns

### Simple name / wildcard

```powershell
es myfile.txt          # exact name anywhere on indexed drives
es "*.log"             # all .log files
es "config*.json"      # name starts with "config", ends with ".json"
```

### Boolean operators

Remember the argv rule from the top of this doc — **spaces inside one quoted
argv become literal substring matches, not operators**. Use either separate
argv tokens or glue without spaces:

```powershell
# OR (three argv)
es "*.cpp" "|" "*.h"
# OR (single argv, no spaces)
es "*.cpp|*.h"

# AND-NOT (two argv)
es "*.log" "!temp"
# AND-NOT (single argv, no spaces)
es "*.log!temp"

# Grouping — easiest as single argv with no spaces
es "<*.cpp|*.h>!build"

# AND (default) — two argv equivalent to substring1 AND substring2 in name
es "README" "md"             # files with both "README" and "md" in name

# Quoted exact phrase (escape inner quotes in PowerShell)
es "\`"exact phrase.txt\`""
es '"exact phrase.txt"'      # easier in single-quoted PS string
```

### Extension filter (fastest — uses the index directly)

```powershell
es "ext:cpp"           # all C++ source files
es "ext:cpp;h"         # C++ sources AND headers (semicolon = OR on extensions)
es "ext:jpg;jpeg;png"  # images
```

### Scope to a directory

```powershell
es -path "C:\Projects\MyApp" "*.cs"
es -path "D:\Logs" "ext:log"
```

`-path` restricts results to files **inside** the given directory tree.
For *parent folder only* (no recursion) use `-parent "C:\dir"`.

### Regex search

```powershell
es -r "test_.*\.py"              # files matching a regex (whole index)
es "regex:test_.*\.py"           # same via prefix syntax
```

**Pitfall**: the CLI flag `-r` **does NOT combine with `-path` or `-p`** —
the combination returns 0 results regardless of the regex. Use the
**prefix-modifier form** instead:

```powershell
# WRONG — silently returns 0
es -r -path "C:\src" "IFoo.*\.h"

# RIGHT — combine prefix modifiers as separate argv
es "path:C:\src" "regex:IFoo.*\.h"
```

### Match against path, not just filename

```powershell
es -p "MyApp\bin"           # matches if "MyApp\bin" appears anywhere in full path
es "path:src\\tests"        # same idea via syntax
```

### Files only / directories only

```powershell
es "/a-d" "*.dll"   # files only (no folders)
es "/ad"  "build"   # directories only
es "file:*.txt"     # alternative syntax
es "folder:src"     # alternative syntax
```

## Filtering by size

Size accepts **bytes, KB, MB, GB, TB** and comparison operators. Ranges use
`..`. Constants are also accepted: `empty`, `tiny`, `small`, `medium`, `large`,
`huge`, `gigantic`.

```powershell
es "size:>2gb"                       # everything bigger than 2 GB
es "size:>2gb" "ext:iso;vmdk"        # 2 GB+ disk images (two argv — argv rule!)
es "size:100mb..1gb"                 # range
es "size:0"                          # zero-byte files
es "size:empty"                      # same, via constant
es -path "D:\" "size:>5gb"           # large files on D:
```

### Sorting by size (largest first)

```powershell
es "size:>1gb" -sort size-descending -size
es -n 20 "ext:mp4" -sort size-descending -size -date-modified
```

`-size`, `-date-modified`, `-dm`, `-dc`, `-da` are **column flags** that add
columns to the output. Without them only the path is printed.

## Filtering by date

Three date axes — modified (`dm:`), created (`dc:`), accessed (`da:`) — plus
"recently changed" (`drc:`). All accept the same vocabulary:

```powershell
es "dm:today"
es "dm:yesterday"
es "dm:thisweek"
es "dm:lastweek"
es "dm:thismonth"
es "dm:2024"                     # any time in 2024
es "dm:2024-01-01..2024-12-31"   # explicit range
es "dc:>2024-06-01"              # created after a date
es "dm:today" "ext:cpp;h"        # combined with other filters (two argv — argv rule!)
```

## Attributes

`attrib:` accepts DIR-style letters: `A` archive, `D` directory, `H` hidden,
`R` readonly, `S` system, `C` compressed, `E` encrypted, `O` offline, `T`
temporary, `L` reparse point, `N` not-indexed.

```powershell
es "attrib:H"          # hidden files
es "attrib:RH"         # readonly AND hidden
es "!attrib:H"         # exclude hidden
```

## Empty: `empty:` vs `size:0`

These are NOT synonyms — a common mistake:

- **`empty:`** matches **folders with no items inside**. On a typical machine
  this can be tens of thousands of empty subfolders.
- **`size:0`** (or `size:empty`) matches **zero-byte files**.

```powershell
es -get-result-count "empty:"           # empty FOLDERS (e.g. 137 671 here)
es -get-result-count "empty:" "/ad"     # same — folders only
es -get-result-count "empty:" "/a-d"    # 0 — empty: doesn't apply to files
es -get-result-count "size:0"           # zero-byte FILES
```

## Duplicates

`dupe:` and friends are **prefix functions** — the argument must be glued
directly to the colon or wrapped in `<...>`. A space after the colon turns it
into a literal search for the word "dupe:" and silently returns zero matches.

```powershell
# WRONG — silently returns 0
es "dupe: ext:mp3"
es "ext:mp3 dupe:"

# RIGHT — glued or grouped
es "dupe:ext:mp3"             # files with same name among .mp3
es "dupe:<ext:mp3>"           # same, with explicit grouping
es "dupe:<size:>100mb>"       # same-name dupes among files >100 MB
es "sizedupe:<ext:iso>"       # files with identical size among .iso
es "sizedupe:<size:>500mb>"   # large files sharing exact byte count
es "dmdupe:<ext:log>"         # same modification date
es "attribdupe:<*.tmp>"       # same attributes
```

To find duplicates of a specific filename across the system, just search for
the name — `dupe:` is only needed when *discovering* duplicates by category:

```powershell
es "CMakeLists.txt"           # all copies of this name (894 across my drives)
```

Sort by name to keep duplicate groups adjacent in the output:

```powershell
es -sort name "dupe:<ext:dll>" -size
```

## Media / file-type macros — REQUIRE GUI SETUP

The macros `audio:`, `video:`, `pic:`, `doc:`, `zip:`, `exe:` are **NOT
built-in** in stock Everything. They are user-defined Filter Lists that must
be configured in **Everything → Tools → Filters**. On an out-of-the-box
install they silently return **0 results**.

**Reliable substitute**: explicit extension lists via the always-built-in
`ext:` prefix. These hit the MFT index directly and are faster anyway:

```powershell
es "ext:mp3;flac;wav;ogg;m4a;aac;wma"           # audio
es "ext:mp4;mkv;avi;mov;wmv;flv;webm"           # video
es "ext:jpg;jpeg;png;gif;bmp;tif;tiff;webp"     # images
es "ext:pdf;doc;docx;xls;xlsx;ppt;pptx;txt;rtf" # documents
es "ext:zip;7z;rar;gz;bz2;xz;tar"               # archives
es "ext:exe;dll;msi;com;bat;cmd"                # executables
```

### Media metadata filters — REQUIRE indexing setup AND can be slow

`dimensions:1920x1080`, `width:>3000`, `height:>2000`, `bitdepth:24`,
`artist:`, `album:`, `title:`, `year:`, `genre:` only work when Everything is
configured to index media properties / audio tags (**Tools → Options →
Folders → Index file properties / Index audio tags**). Without that setup
these queries **fall back to content-scanning files**, which can hang for
seconds-to-minutes and read every matching file from disk.

If you need media metadata, verify in Everything's Options dialog first; if
it isn't enabled, use `ext:` + `size:` heuristics instead.

## Limiting & paginating results

```powershell
es -n 20 "*.tmp"                # first 20 matches
es -o 100 -n 20 "*.tmp"         # results 101..120
es -get-result-count "*.cpp"    # just the number, no listing
es -get-total-size "size:>2gb"  # sum of file sizes in bytes, no listing
```

## Sorting

```powershell
es "*.log" -sort date-modified-descending
es "*.iso" -sort size-descending
es "ext:cpp" -sort path-ascending
```

Valid keys: `name`, `path`, `size`, `extension`, `date-created`,
`date-modified`, `date-accessed`, `attributes`, `run-count`,
`date-recently-changed`, `date-run`. Each accepts `-ascending` / `-descending`
suffix; or set direction globally with `-sort-ascending` / `-sort-descending`.

## Output format

### Default

One full path per line on stdout. Add column flags to enrich. **Column order
is fixed by es**: extra columns (size, dates, attributes, ext) are printed
**before** the path, separated by whitespace:

```powershell
es "size:>1gb" -size -dm -sort size-descending
# 4,608,819,200 13/04/2022 12:10 Z:\HYPER-V\ISO\Windows.iso
#  ^size         ^modified date  ^path (always last)
```

### Structured export

```powershell
es "*.sln" -csv                          # CSV to stdout
es "*.sln" -tsv                          # TSV to stdout (safer than CSV for paths)
es "*.sln" -export-csv results.csv       # CSV to file
es "*.sln" -export-tsv results.tsv       # TSV to file
es "*.sln" -export-efu results.efu       # Everything's native format
es "ext:mp3" -export-m3u8 playlist.m3u8
```

**Column behavior**:
- `-csv` / `-tsv` (stdout and `-export-csv`/`-export-tsv`) output **only the
  Filename column by default**. To include more, pass column flags
  explicitly: `-size -dm -dc -ext -attributes`.
- `-efu` (stdout and `-export-efu`) **always** emits a fixed set:
  Filename, Size, Date Modified, Date Created, Attributes — column flags are
  ignored.

```powershell
# CSV with size + modification date
es "ext:iso" -csv -size -dm -sort size-descending
```

EFU, TSV, and CSV are easy to parse from PowerShell with
`Import-Csv` (use `-Delimiter \"`t\"` for TSV).

### Format tweaks

```powershell
es ... -size-format 1        # 0=auto, 1=bytes, 2=KB, 3=MB
es ... -date-format 1        # 0=system, 1=ISO-8601, 2=FILETIME, 3=ISO-UTC
es ... -no-digit-grouping    # no thousands separators
es ... -no-header            # omit CSV header
```

## Case sensitivity — `-i` IS NOT ignore-case!

In `es.exe`, the `-i` flag means **match-case** (case-sensitive), which is the
**opposite** of POSIX `grep -i`. The default is already case-insensitive.
Three equivalent ways to enable case sensitivity:

```powershell
es "CLAUDE.md"           # case-insensitive (default) — finds CLAUDE.md and claude.md
es -i "CLAUDE.md"        # case-SENSITIVE — only exact-case matches
es -case "CLAUDE.md"     # same: case-sensitive
es "case:CLAUDE.md"      # same: case-sensitive via prefix
```

Whole-word matching has three equivalent forms:

```powershell
es -w "claude"           # whole word
es -ww "claude"          # synonym
es "wholeword:claude"    # prefix form
```

## Combined examples

Remember the argv rule — each search token is a separate argv, or glue them
without spaces.

```powershell
# Top-10 biggest files on the system
es -n 10 -sort size-descending -size "size:>1gb"

# All ISO images larger than 2 GB on D: drive
es -path "D:\" -size -sort size-descending "size:>2gb" "ext:iso"

# Source files modified today in current project
es -path "Z:\PROJECTS\MyApp" -dm "dm:today" "ext:cpp;h;qml"

# Hidden files in user profile (files only, hidden attribute)
es -path "$env:USERPROFILE" "attrib:H" "/a-d"

# Find all duplicate .dll names across all drives, grouped together
es -sort name -size "dupe:<ext:dll>"

# Find large files (>500 MB) that share an exact byte count — strong dedupe hint
es -sort size-descending -size "sizedupe:<size:>500mb>"

# Empty folders inside a project tree
es -path "C:\Projects" "empty:"

# 50 largest videos, exported to CSV with size & date columns
es -sort size-descending -n 50 -size -dm `
   -export-csv "$env:TEMP\big-videos.csv" `
   "ext:mp4;mkv;avi;mov" "size:>500mb"

# How much disk space is taken by files >2 GB total? (bytes)
es -get-total-size "size:>2gb"
```

## Interpreting exit codes

| Code | Meaning |
|------|---------|
| 0 | Success — including the "0 results" case unless `-no-result-error` is set |
| 6 | Unknown switch (e.g. GNU-style `--version`); help is still printed to stderr |
| 8 | Everything service is not running — fall back |
| 9 | No results — only when `-no-result-error` is passed |

## Escaping literal special chars in filenames (`^`)

To search for filenames that **literally contain** `\`, `&`, `|`, `>`, `<`, or
`^`, prefix the character with `^` inside the search syntax. This is
*orthogonal* to the argv-tokenization rule above — `^` doesn't affect how
spaces are parsed, it only neutralizes operator chars so they match as
literals.

```powershell
# Find files whose name contains a literal pipe character
es "^|"

# Find files whose name contains "AT&T"
es "AT^&T"

# Find files whose name contains "<draft>"
es "^<draft^>"
```

Note: `*.cpp^|*.h` does **not** mean "OR" — the `^` makes `|` literal, so this
searches for filenames containing the substring `*.cpp|*.h`. To get OR, use
the argv rule from the top of this doc.

## PowerShell quoting tips

- Wrap each search token in **double quotes** when it contains `<>`, `|`, or
  `!` — PowerShell otherwise parses them as redirection / pipeline /
  history-expansion operators.
- Inside a double-quoted PS string, embedded `"` must be backtick-escaped:
  `es "\`"exact phrase\`""`. Single-quoted PS strings need no escape:
  `es '"exact phrase"'`.
- For literal strings without variable expansion, prefer single quotes:
  `es 'size:>2gb' '!attrib:H'`.
- The pipe `|` inside an Everything OR query MUST be inside a quoted token —
  either as part of a no-space token `"*.cpp|*.h"` or as its own argv `"|"`.
  Bare `|` is interpreted by PowerShell as a pipeline operator.
- **The argv rule (see top of doc) overrides the natural-syntax temptation**:
  do NOT cram a multi-operator query into one quoted string with spaces. It
  becomes a literal substring search. Split into separate quoted argv tokens,
  or remove the spaces around operators.

## Fallback when es.exe is unavailable

If `es.exe` is not installed or Everything is not running, use these
equivalents:

| Goal | Fallback |
|------|----------|
| Find files by name | `Glob` tool with a `**/*.ext` pattern |
| Find in specific dir | `Get-ChildItem -Recurse -Filter "*.ext" -Path "C:\dir"` |
| Count files | `(Get-ChildItem -Recurse -Filter "*.ext").Count` |
| Regex name match | `Get-ChildItem -Recurse \| Where-Object Name -Match "pattern"` (PS `Match` is case-insensitive by default) |
| Size filter | `Get-ChildItem -Recurse \| Where-Object Length -gt 2GB` |
| Modified today | `Get-ChildItem -Recurse \| Where-Object { $_.LastWriteTime -gt (Get-Date).Date }` |

Note: when falling back, remember `es.exe`'s `-i` is **match-case** (the
opposite of PowerShell's `-CMatch` / `grep -i`). Don't carry the `-i`
shorthand into the fallback commands.

Always tell the user which tool you fell back to and why, so they know to start
Everything if they want faster results next time.

## What es-search does NOT do well

- **Content search** (text inside files) → use the `Grep` tool. Everything
  *does* have `content:` syntax, but it bypasses the index and is slow.
- **Directory listing** (not recursive) → use `ls` / `Get-ChildItem`.
- **Cross-platform search** → `es.exe` is Windows-only; use `Glob` on
  macOS/Linux.
- **Non-indexed locations** (network shares not added to Everything, some
  removable media) — Everything only sees what its index covers.

---

*Licensed under the [MIT License](LICENSE).*
