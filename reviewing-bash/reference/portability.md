# Portability: bash versions, POSIX sh, BSD vs GNU

A bash script that works on the author's machine may not work on
deployment. Three axes matter:

1. **Bash dialect vs POSIX sh.** Different feature sets.
2. **Bash version.** 3.2 (macOS default), 4.0, 4.3, 4.4, 5.0, 5.1+.
   Major features arrived in each.
3. **GNU vs BSD coreutils / findutils.** `find`, `sed`, `awk`,
   `date`, `readlink`, `mktemp`, `stat`, `xargs` diverge.

Reviewers should know what the deployment target is. If the script
runs only on the author's box, none of this matters. If it runs in
CI, in containers, in production, on macOS, in BusyBox, on
Termux, the constraints multiply.

## Contents

- Shebang and dialect choice
- shellcheck dialect (`-s sh` vs `-s bash`)
- Bash version reference
- POSIX sh restrictions
- GNU vs BSD divergences
- macOS-specific gotchas
- BusyBox / Alpine considerations
- Reviewer checklist

## Shebang and dialect choice

| Shebang | Interpreter | Feature set |
|---------|-------------|-------------|
| `#!/bin/sh` | system `sh` (dash on Debian, ash on Alpine, bash-in-POSIX on RHEL, bash on macOS) | POSIX only |
| `#!/usr/bin/env bash` | first bash in `$PATH` | Bash, whatever version |
| `#!/bin/bash` | the bash at `/bin/bash` | Bash, may be missing on \*BSD/Alpine |
| `#!/usr/bin/env -S bash -euo pipefail` | bash with flags (Linux coreutils 8.30+) | Bash, with strict-mode preamble in shebang |

Source: Google
[s1.1](https://google.github.io/styleguide/shellguide.html#s1.1-which-shell-to-use),
Wooledge
[BashGuide/Practices](https://mywiki.wooledge.org/BashGuide/Practices).

When `#!/bin/sh`, bash-isms become real bugs even if `/bin/sh` is a
bash symlink (because bash in POSIX mode disables them). If a script
uses `[[ ]]`, arrays, `<<<`, `${var^^}`, `${var/x/y}`, etc., the
shebang must be bash, not sh.

## shellcheck dialect

shellcheck picks the dialect from the shebang. Override with
`-s sh|bash|dash|ksh`. Worth knowing:

- `-s bash`: tolerates `[[ ]]`, `==`, `function`, arrays, etc.
- `-s sh` / `-s dash`: flags those as non-POSIX (`SC3xxx` codes).

Examples that change behaviour:

```bash
# Under -s bash: nothing flagged.
# Under -s sh: SC3014 (== non-POSIX), SC3010 ([[ non-POSIX), etc.
if [[ $x == foo ]]; then echo y; fi
```

If a script has `#!/bin/sh` but uses bash features, shellcheck's
SC3xxx warnings are real portability bugs, not stylistic notes. If
the script has `#!/usr/bin/env bash`, ignore SC3xxx.

## Bash version reference

Each major version added features. Older systems (macOS, embedded,
ancient enterprise distros) lag.

| Feature | Bash version | Notes |
|---------|--------------|-------|
| `[[ ]]`, `(( ))` | 2.x | core conditionals; safe to assume everywhere bash exists |
| Indexed arrays | 2.x | core |
| `$(< file)` | 2.x | builtin substitute for `$(cat file)` |
| `<<<` (here-strings) | 2.05b | safe to assume |
| `printf -v` | 3.1 | assign without subshell |
| `+=` for strings/arrays | 3.1 | append |
| `${var//pat/repl}` (replace-all) | 3.x | core parameter expansion |
| `function f { … }` and `f() { … }` | 2.x | both forms exist; mixing is non-portable |
| `coproc` | 4.0 | two-way pipes |
| `mapfile` / `readarray` | 4.0 | read file into array |
| Associative arrays (`declare -A`) | 4.0 | key-value |
| `${var^^}` / `${var,,}` (case conversion) | 4.0 | parameter expansion |
| `**` recursive glob (`globstar`) | 4.0 | `shopt -s globstar` |
| `declare -n` (name references) | 4.3 | indirect (has injection caveats) |
| `wait -n` | 4.3 | wait for any one job |
| `shopt -s inherit_errexit` | 4.4 | `set -e` propagates into `$(…)` |
| `mapfile -d` (delimiter) | 4.4 | NUL-delimited reads |
| `${var@Q}` (shell-quote) | 4.4 | safe representation |
| `wait -f` (wait for `fg` jobs) | 5.1 | |
| `EPOCHSECONDS`, `EPOCHREALTIME` | 5.0 | no `date` fork |
| `SRANDOM` (better RNG) | 5.1 | |

Find a script's required version with `bash --version` on the
deployment box, or grep for the listed features and pick the highest
version they require.

### macOS bash 3.2

macOS ships bash 3.2 (last release 2007) and won't update beyond it
because of GPL v3. So on default-macOS, **none of bash 4+ exists**:

- No associative arrays.
- No `mapfile` / `readarray`.
- No `${var^^}` / `${var,,}`.
- No `globstar`.
- No `coproc`.

A script that targets macOS-default must stay below the bash-4.0
line. Workaround: install bash 4+ via Homebrew (`brew install bash`)
and require it via shebang (`#!/usr/bin/env bash` will pick the
Homebrew one if `/usr/local/bin` or `/opt/homebrew/bin` is in PATH
ahead of `/bin`).

## POSIX sh restrictions

If the shebang is `#!/bin/sh`, the script must avoid:

- `[[ ]]` (use `[ ]`).
- Arrays of any kind.
- `<<<` here-strings.
- `(( ))` arithmetic (use `$(( ))` for expansion, `expr` for
  side-effecting math, or use `:` + arithmetic).
- `function` keyword (use `name() { … }`).
- `==` in `[ ]` (use `=`). SC3014.
- `${var/pat/repl}`, `${var^^}` etc. parameter expansion beyond
  `${var:-x}` / `${var:+x}` / `${var%x}` / `${var#x}`.
- `source` (use `.`).
- `local` (POSIX has no `local`; many shells provide it as
  extension, but not all).
- `$RANDOM`, `$SECONDS`, `$EPOCHSECONDS`.
- Process substitution `< <(cmd)`.
- `pipefail`, `errtrace` (POSIX has `errexit` and `nounset` only).
- Brace expansion `{a,b,c}` and `{1..10}`.

Useful approximations in POSIX:

```sh
# associative-array-like with eval (ugly, but POSIX)
eval "tbl_$key=value"
eval "value=\$tbl_$key"

# array-like with positional parameters
set -- a b c
for x; do echo "$x"; done

# string lower-case (no parameter expansion)
echo "$x" | tr '[:upper:]' '[:lower:]'

# basename/dirname (no parameter expansion)
basename "$path"
dirname "$path"
```

If the script needs more than a sequence of utility calls, leave
POSIX sh and go to bash (or further).

## GNU vs BSD divergences

These tools have the same names but different flag sets and
defaults.

### find

| GNU `find` | BSD/macOS `find` | Notes |
|------------|-------------------|-------|
| `-printf '%s %p\n'` | no equivalent | use `-exec stat …` or pipe to `stat` |
| `-iname` | `-iname` | both |
| `-type f` | `-type f` | both |
| `-print0` | `-print0` | both |
| `-execdir` | `-execdir` | both |
| `-regex` | `-E -regex` | GNU regex POSIX-ish; BSD needs `-E` for ERE |
| `-not` | `\! …` | both accept `-not`, but BSD is finicky |
| `-empty` | `-empty` | both |
| `-newer` | `-newer` | both |
| `-mtime +N` | `-mtime +N` | both |
| `-maxdepth N` | `-maxdepth N` | both (BSD got it later but it's there) |

A script using `find -printf` won't work on macOS.

### sed

- **In-place edit:** GNU `sed -i 's/x/y/' file` vs
  BSD `sed -i '' 's/x/y/' file` (BSD requires an explicit suffix
  argument, even if empty).
- **Extended regex:** GNU `sed -E` (modern) or `-r` (legacy);
  BSD `sed -E`. Both accept `-E` now.
- **`\b` word boundary, `\d` digit:** GNU sed treats them as
  literals or as extensions; BSD doesn't support them. Use POSIX
  classes (`[[:alpha:]]`).

Cross-platform in-place edit:

```bash
sed -i.bak 's/x/y/' file && rm file.bak
```

Or use `perl -i -pe`.

### awk

`awk` is fairly portable (POSIX has a spec), but GNU awk extends it
heavily. Watch for:

- `gensub`, `length(array)`, multidimensional arrays: gawk-only.
- `RS=""` (paragraph mode): POSIX awk doesn't support arbitrary
  multi-char `RS`.
- `getline` semantics vary.

For complex awk, depend on gawk explicitly (`/usr/bin/env gawk`) or
keep to POSIX awk.

### date

- **GNU `date -d 'next week'`** (relative dates): GNU only.
- **BSD `date -v +1w`** (relative dates with `-v`): BSD only.
- **`date +'%(%Y…)T'`** format codes: same on both, but BSD lacks
  some specifiers.
- **`%N` (nanoseconds):** GNU only.

For cross-platform, install GNU coreutils on macOS (`brew install
coreutils`, then `gdate`), or use bash's `printf '%(…)T'` (bash 4.2+).

### readlink

- **GNU `readlink -f`** (canonicalize): GNU only.
- **BSD `readlink`** has no `-f` until macOS 12.3 (Monterey).
- Portable replacement: `python3 -c 'import os,sys;print(os.path.realpath(sys.argv[1]))'`
  or a loop in bash.

### mktemp

- **GNU `mktemp -d`**: works, default template OK.
- **BSD `mktemp -d`**: works, but the template must be explicit:
  `mktemp -d -t prefix`.
- Cross-platform: `mktemp -d 2>/dev/null || mktemp -d -t tmp`.

### stat

- **GNU `stat -c '%s' file`** (custom format): GNU only.
- **BSD `stat -f '%z' file`** (custom format): BSD only, different
  specifier.
- Cross-platform: `wc -c < file` for size.

### xargs

- **GNU `xargs -d '\n'`**: GNU only.
- **BSD `xargs`**: no `-d`; use `-0` with `-print0` from find.
- Both support `-I {}`, `-n N`, `-P N`, `-0`.

## macOS-specific gotchas

In addition to bash 3.2 and BSD tools:

- `/usr/bin/python` is gone (Catalina+). Use `/usr/bin/env python3`
  if you must invoke Python.
- `realpath` from `coreutils` is not preinstalled. Same workarounds
  as `readlink -f`.
- `tac` (reverse cat): GNU. Use `tail -r` on BSD/macOS.
- `seq`: BSD has it; check formatting.
- Case-insensitive filesystem by default (HFS+/APFS): scripts that
  rely on case sensitivity in filenames break.

## BusyBox / Alpine considerations

Alpine Linux ships BusyBox utilities. Most flags are present but
some are missing or subtly different. The shell is `ash`, not bash;
shebang `#!/bin/sh` runs ash.

Common BusyBox surprises:

- `find` is mostly GNU-compatible but lacks some flags.
- `sort -h` (human-numeric): BusyBox supports it; older Alpine may
  not.
- `date -d` and `date -r`: BusyBox supports both.
- `awk` is BusyBox awk, a tiny POSIX-ish implementation; gawk-isms
  fail.
- `grep -P` (Perl regex): missing.
- `xargs -P` (parallel): missing or limited.

If the script is for an Alpine container, install `bash` and
`coreutils` explicitly in the Dockerfile, or stay within BusyBox's
feature set.

## Reviewer checklist

1. **Read the shebang.** Sets the dialect ceiling.
2. **Identify deployment targets.** CI runner OS, container base
   image, macOS dev boxes, production servers.
3. **Match features against the targets.** Any feature above the
   floor (oldest target's bash version, BSD vs GNU) is a portability
   bug.
4. **Run shellcheck with the matching dialect.** `-s sh` if shebang
   is `/bin/sh`; let it auto-pick otherwise.
5. **Spot-check the GNU-only tools.** `find -printf`, `sed -i`,
   `date -d`, `readlink -f`, `stat -c`, `xargs -d` are the most
   common cross-platform breakages.

## Sources

- Wooledge
  [BashWeaknesses](https://mywiki.wooledge.org/BashWeaknesses) (on
  bash's general limits),
  [BashFAQ/061](https://mywiki.wooledge.org/BashFAQ/061) (which
  features in which bash version).
- Google
  [s1.1](https://google.github.io/styleguide/shellguide.html#s1.1-which-shell-to-use).
- ShellCheck SC3xxx codes: POSIX portability.
- Apple's bash situation: bash 3.2 ships; bash 5 via Homebrew. zsh
  is the macOS default *interactive* shell since Catalina but `/bin/sh`
  and `/bin/bash` still exist (bash 3.2).
- Alpine / BusyBox utility differences: per the BusyBox manpages.
