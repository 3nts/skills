# Example review walkthrough

One full review end-to-end, with real `shellcheck` 0.11.0 and `shfmt`
v3.13.1 output. The script and the diagnostics are not invented; this
is what the tools actually produce. Use this as the format model for
your own reviews.

The skill's `Output format` section in SKILL.md prescribes:

1. Findings list, grouped by impact tier.
2. The refactored script (only if a refactor was requested).

What follows demonstrates that shape, including the cases where tool
output and reviewer judgment disagree.

## Contents

- The script
- Step 1: read in full
- Step 2: run the tools
- Step 3: grouped findings
- Step 4: refactored version
- Appendix: clean-script and stylistic-only scenarios

## The script

```bash
#!/bin/sh
LOG_DIR=/var/log/myapp
files=`ls $LOG_DIR/*.log`
for f in $files
do
  if [ $f == "/var/log/myapp/error.log" ]
  then
    echo Found error log: $f
    cat $f | grep -v "^#" > $f.clean
  fi
done

cd $1
rm -rf *.tmp
```

User prompt: *"this script is ugly, what's wrong with it?"*

## Step 1: read in full

Quick mental model before commenting:

- Shebang is `#!/bin/sh` (POSIX), so bash-only features are off the
  table even if bash is the resolved interpreter.
- The script finds `*.log` files under a hardcoded directory, picks
  out `error.log`, strips comment lines into `<f>.clean`.
- Then it `cd`s to `$1` and deletes `*.tmp` files there. The `cd` is
  unchecked. The script as-written may delete tmp files in the
  caller's CWD if `cd` fails.
- Inputs: `$1` (target dir). Hardcoded: `LOG_DIR`. No error handling.

## Step 2: run the tools

```
$ shellcheck buggy.sh
In buggy.sh line 3:
files=`ls $LOG_DIR/*.log`
      ^-----------------^ SC2006 (style): Use $(...) notation instead of legacy backticks `...`.

In buggy.sh line 6:
  if [ $f == "/var/log/myapp/error.log" ]
       ^-- SC2086 (info): Double quote to prevent globbing and word splitting.
          ^-- SC3014 (warning): In POSIX sh, == in place of = is undefined.

In buggy.sh line 8:
    echo Found error log: $f
                          ^-- SC2086 (info): Double quote to prevent globbing and word splitting.

In buggy.sh line 9:
    cat $f | grep -v "^#" > $f.clean
        ^-- SC2086 (info): Double quote to prevent globbing and word splitting.
                            ^-- SC2086 (info): Double quote to prevent globbing and word splitting.

In buggy.sh line 13:
cd $1
^---^ SC2164 (warning): Use 'cd ... || exit' or 'cd ... || return' in case cd fails.
   ^-- SC2086 (info): Double quote to prevent globbing and word splitting.

In buggy.sh line 14:
rm -rf *.tmp
       ^-- SC2035 (info): Use ./*glob* or -- *glob* so names with dashes won't become options.
```

```
$ shfmt -i 2 -ci -sr -bn -d buggy.sh
@@ -1,10 +1,8 @@
 #!/bin/sh
 LOG_DIR=/var/log/myapp
-files=`ls $LOG_DIR/*.log`
-for f in $files
-do
-  if [ $f == "/var/log/myapp/error.log" ]
-  then
+files=$(ls $LOG_DIR/*.log)
+for f in $files; do
+  if [ $f == "/var/log/myapp/error.log" ]; then
     echo Found error log: $f
     cat $f | grep -v "^#" > $f.clean
   fi
```

Tools caught the mechanical issues. Things they missed (because they
require seeing the script as a whole, not as isolated patterns):

- The `ls` into a `for` loop is BashPitfalls/1; `files=$(ls …)`
  then `for f in $files` is the same bug after shfmt's substitution
  rewrite. shellcheck doesn't flag this form, but SC2045 fires for
  `for f in $(ls)` directly. Worth flagging either way.
- The `rm -rf *.tmp` after an unchecked `cd "$1"` is destructive in
  the wrong CWD. shellcheck's SC2164 catches the unchecked cd; SC2035
  catches the bare glob; but the compound consequence is reviewer
  judgment.
- `cat $f | grep …` is UUOC ([SC2002](https://www.shellcheck.net/wiki/SC2002)),
  but SC2002 is opt-in (`shellcheck -o useless-use-of-cat`). It's
  worth fixing for performance even though default shellcheck stays
  silent.

## Step 3: grouped findings

```
Findings: buggy.sh

REAL BUGS
  L13 | cd $1 unchecked; if cd fails, rm runs in caller's CWD
      | fix: cd "$1" || exit
      | SC2164, SC2086
  L14 | rm -rf *.tmp; with cd failure above, deletes wrong tree
      | fix: previous fix + rm -rf -- ./*.tmp
      | SC2035; compound risk
  L6  | == in POSIX [ ] is undefined; shebang is /bin/sh
      | fix: use = (or switch shebang to /usr/bin/env bash)
      | SC3014, BashPitfalls/20

FRAGILITY
  L3  | files=`ls ...` then for f in $files word-splits names with spaces
      | fix: for f in "$LOG_DIR"/*.log
      | BashPitfalls/1, SC2006 (backticks separately)
  L6,8,9,13 | unquoted parameter expansion throughout
      | fix: quote every "$var"
      | SC2086 (five occurrences)

IDIOMS
  L9  | cat "$f" | grep ... is UUOC
      | fix: grep -v "^#" "$f" > "$f.clean"
      | SC2002 (opt-in); also one fewer fork

STYLE
  L3  | backticks
      | fix: $()
      | SC2006, Google s6.2, BashFAQ/082
  L4-5, L6-7 | do/then on separate lines
      | fix: ; do / ; then
      | Google s5.4, shfmt -bn
```

## Step 4: refactored version

```bash
#!/usr/bin/env bash
# Strip comment lines from each *.log under LOG_DIR; clean up tmp files
# under the caller-supplied target.

set -eu

LOG_DIR=/var/log/myapp
TARGET=${1:?usage: $0 <target-dir>}

shopt -s nullglob

for f in "$LOG_DIR"/*.log; do
  if [[ $f == "$LOG_DIR/error.log" ]]; then
    printf 'Found error log: %s\n' "$f"
    grep -v '^#' "$f" > "$f.clean"
  fi
done

cd "$TARGET" || exit 1
rm -rf -- ./*.tmp
```

What changed and why:

- Shebang `#!/usr/bin/env bash`: opens up `[[ ]]`, brace expansion,
  and quoted-variable conveniences. POSIX sh isn't load-bearing here.
- `set -eu` (not `pipefail`, no `IFS` reset): the script has no
  pipelines, and we add a `nullglob` rather than relying on `IFS`
  word splitting. This is a debatable choice; see
  [strict-mode.md](strict-mode.md).
- `TARGET=${1:?usage: …}`: explicit required-arg failure with a
  message instead of silent `cd ""` confusion.
- `shopt -s nullglob`: makes the glob expand to nothing rather than
  to the literal `*.log` if no matches.
- `for f in "$LOG_DIR"/*.log`: direct glob, no `ls`, no word
  splitting on filenames.
- `[[ ]]` with `=` for literal compare (RHS quoted is irrelevant
  here because there's no glob in the RHS, but quoted is the safer
  default).
- `printf '%s\n'` over `echo "Found …"`: avoids any `echo`
  weirdness with the filename.
- `grep -v '^#' "$f"`: direct grep, no cat.
- `cd "$TARGET" || exit 1`: checked.
- `rm -rf -- ./*.tmp`: `--` ends options; `./` prevents a file
  named `-rf` (hypothetical) from being interpreted as flags.

## Appendix: clean-script and stylistic-only scenarios

### Clean script

```bash
#!/usr/bin/env bash
# Print files matching pattern, sorted by name.

set -euo pipefail

list_files() {
  local dir=${1:-.}
  local pattern=${2:-*}
  find "$dir" -type f -name "$pattern" | sort
}

list_files "$@"
```

Real tool output:

```
$ shellcheck -s bash clean.sh
(no output, exit 0)

$ shfmt -i 2 -ci -sr -bn -d clean.sh
(no diff, exit 0)
```

Correct review output: **"Looks fine. shellcheck and shfmt are
clean; quoting and strict-mode pairing are appropriate for a
short read-only script. Nothing to flag."**

A bad reviewer manufactures findings here ("could rename `dir` to
`directory`", "could add error message"). Don't. Wooledge's
[BashGuide/Practices](https://mywiki.wooledge.org/BashGuide/Practices)
and the SKILL.md "push back" rule both apply.

### Stylistic-only script

```bash
#!/usr/bin/env bash
function process_record()
{
    local record="$1"
    local count="$2"
    if [[ "$record" == "header" ]]
    then
        printf 'Header: %s (count=%d)\n' "$record" "$count"
    else
        printf 'Other: %s\n' "$record"
    fi
}
process_record "header" 5
```

Real tool output:

```
$ shellcheck -s bash stylistic.sh
(no output, exit 0)

$ shfmt -i 2 -ci -sr -bn -d stylistic.sh
@@ -1,13 +1,11 @@
 #!/usr/bin/env bash
-function process_record()
-{
-    local record="$1"
-    local count="$2"
-    if [[ "$record" == "header" ]]
-    then
-        printf 'Header: %s (count=%d)\n' "$record" "$count"
-    else
-        printf 'Other: %s\n' "$record"
-    fi
+function process_record() {
+  local record="$1"
+  local count="$2"
+  if [[ "$record" == "header" ]]; then
+    printf 'Header: %s (count=%d)\n' "$record" "$count"
+  else
+    printf 'Other: %s\n' "$record"
+  fi
 }
 process_record "header" 5
```

Correct review output: report shfmt's diff as the recommendation
(2-space indent, brace on same line, `; then` consolidated). Note
separately that `function name() { … }` combines `function` and
`()` and is non-portable
([BashPitfalls/25](https://mywiki.wooledge.org/BashPitfalls#pf25));
pick one form. shellcheck does not flag this under `-s bash` because
bash accepts both; it's a Wooledge / Google convention, not a
shellcheck rule.

No real bugs. Don't claim there are any.

## Notes on tool output that diverged from the SKILL.md model

- **shellcheck severity:** the `(style)`, `(info)`, `(warning)`,
  `(error)` annotations in shellcheck output map roughly to the
  skill's "stylistic / fragility / real-bug" tiers, but not 1:1.
  SC2086 is `(info)` even though unquoted expansion is often a real
  bug; SC2164 is `(warning)` because the bug is conditional on `cd`
  failing.
- **shfmt converts backticks unconditionally,** even without `-s`
  (simplify). Reviewers running shfmt should expect this and not
  call it out separately if shellcheck's SC2006 is already on the
  list.
- **Opt-in shellcheck checks** worth knowing about:
  `useless-use-of-cat` (SC2002), `deprecate-which`,
  `require-double-brackets`, `check-set-e-suppressed`,
  `check-unassigned-uppercase`, `add-default-case`,
  `avoid-nullary-conditions`, `quote-safe-variables`,
  `require-variable-braces`. Run `shellcheck --list-optional` to
  see the full set; enable with `-o name` or `-o all`.

## Related

- [strict-mode.md](strict-mode.md): the `set -e`/`-u`/`pipefail` debate
  that shaped the refactor's choices.
- [portability.md](portability.md): the POSIX-vs-bash and BSD-vs-GNU
  considerations that decided the shebang switch.
- [correctness.md](correctness.md): the principles each finding cites.
