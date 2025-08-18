# Filename “sed” — concept

A cross-platform CLI that applies regex (PCRE) and small transforms to file **components** (dir, basename, stem, ext) and can also **move** files based on captures. Safe by default (dry-run, transactional, undo).

## Command name

`pathsed` (aka `fnsed`, `renx`, or `pcrename`)

## Core model

Each file has components:

* `$P` full path
* `$D` directory
* `$B` basename (e.g., `report.final.pdf`)
* `$S` stem (basename without extension, `report.final`)
* `$E` extension without dot (`pdf`)
* `$R` root (drive on Windows, `/` on POSIX)

Operations target one component at a time, or the whole path.

## Syntax (sed-like)

```
pathsed [findspec] s/<regex>/<replacement>/<flags> [--on <component>] [options...]
```

* `<regex>` supports PCRE (named groups, lookaround, Unicode).
* `<replacement>` supports backrefs `\1`, `\g<name>`, and component tokens `{D,S,E,B,P}`.
* `<flags>` include `g` (global), `i` (case-insensitive), `m`, `u`.
* `--on` chooses which component is matched & replaced: `stem|ext|basename|dir|path`. Default: `basename`.
* Multiple `s///` rules apply in order.

### Filters / target selection

* `findspec`: glob(s) or `--glob`/`--ext jpg,png` or `--re '**/*.pdf'`.
* `--where 'stem ~= /regex/'` (mini filter DSL).
* `--type file|dir|symlink` (what to touch).

### Transforms (post-regex helpers)

* `:lower`, `:upper`, `:title`, `:slug`, `:ascii`, `:trim`, `:pad(n)`, `:counter(start,step)`.
* Apply via `--apply 'stem:slug,stem:lower'` or inline replacement functions `{{slug(\1)}}`.

### Moves / templated destinations

* `--move-to '<template>'` where template uses tokens and captures:

  * Tokens: `{D}`, `{S}`, `{E}`, `{B}`, `{R}`, `{year}`, `{month}`, etc. (from regex named groups).
  * Example: `--move-to '{D}/{year}/{month}/{S}.{E}'`.

### Safety & UX

* Dry run by default; require `--write` to commit.
* Collision policy: `--on-collision skip|overwrite|number` (default `number`: adds `-1`, `-2`, …).
* Transaction & undo: writes a journal file and supports `--undo <journal.json>`.
* Case-insensitive FS detection (macOS/Windows), Unicode normalization (`--nfc|--nfd`), reserved names on Windows guarded.
* Progress, TTY preview table, `--json` plan output for scripting.

### Examples

1. Pad numbers in stems to 3 digits:

```
pathsed '**/*' s/(\D*)(\d+)/\1{{pad(\2,3)}}/g --on stem
```

2. Normalize spaces/dashes and lowercase extensions:

```
pathsed '**/*' s/[ _]+/-/g --on stem \
        s/.*/{{lower(\0)}}/ --on ext
```

3. Extract date and move into YYYY/MM folders:

```
pathsed '**/*' s/.*?(?<year>20\d{2})[._-]?(?<month>\d{2})[._-]?(?<day>\d{2}).*/\g<0>/ \
  --on stem \
  --move-to '{D}/{year}/{month}/{S}.{E}' --write
```

4. Collapse multi-extensions like `.tar.gz` into stem + full ext:

```
pathsed '**/*.tar.gz' s/(.*)\.tar\.gz/\1/ --on basename \
  --apply 'stem:slug' \
  --move-to '{D}/{S}.tar.gz' --write
```

5. Title-case music files and reorder “Artist - Track”:

```
pathsed '**/*.mp3' s/^(?<artist>.+)\s*-\s*(?<track>.+)$/\g<track> - \g<artist>/ --on stem \
  s/.*/{{title(\0)}}/ --on stem --write
```

### Implementation sketch

* **Language:** Python with the `regex` module (PCRE-like), or Rust/Go for speed.
* **Planner:** Build a list of `(old_path, new_path)` plus ensured parent dirs. Detect collisions & self-moves.
* **Executor:** Two-phase: create dirs, rename/move files. Journal every step.
* **Finder:** Use `glob` + optional fast walker (e.g., `scandir`) with ignore rules (`.gitignore`, `--ignore`).
* **Cross-platform quirks:** Max path lengths (Windows long path prefix), reserved names, permission errors.

### CLI shape (docopt/argparse)

```
Usage:
  pathsed <findspec>... s/<regex>/<replacement>/<flags>...
          [--on=<component>] [--move-to=<template>]
          [--apply=<transforms>] [--on-collision=<policy>]
          [--dry-run | --write] [--json] [--undo=<journal>]
          [--ext=<list>] [--type=<t>] [--nfc|--nfd]
```

### Tests you’ll want

* Unicode filenames (NFC vs NFD).
* Case-only renames on case-insensitive FS.
* Name collisions and atomic numbering.
* Very deep trees and 100k+ files (perf).
* Symlink safety (rename link vs target).
* Windows reserved names and colon characters.
