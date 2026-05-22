# Bash correctness reference

## Contents

- Quoting
- Test commands and conditionals
- Subshells and pipelines
- Reading input
- Substitutions: `$()` and `printf`
- Arithmetic
- `cd` safety
- Error handling: the strict-mode tension
- eval and code-as-data
- Race conditions and temp files
- Signal handling and traps

## Quoting

Quoting is the single largest source of bash bugs.

- **Quote every parameter expansion.** Unquoted `$var` undergoes word
  splitting on `IFS` and pathname expansion. Use `"$var"`. Source:
  [SC2086](https://www.shellcheck.net/wiki/SC2086),
  [BashPitfalls/14](https://mywiki.wooledge.org/BashPitfalls#pf14),
  [Google s5.7](https://google.github.io/styleguide/shellguide.html#s5.7-quoting).
- **`"${arr[@]}"` for arrays.** Unquoted `${arr[@]}` splits each
  element again; `${arr[*]}` joins on the first char of `IFS`. The
  quoted `[@]` form is almost always what you want. Source:
  [SC2068](https://www.shellcheck.net/wiki/SC2068),
  [BashPitfalls/24](https://mywiki.wooledge.org/BashPitfalls#pf24).
- **`"$@"` for positional parameters.** Same rule:
  [SC2048](https://www.shellcheck.net/wiki/SC2048).
- **Inside `[[ ]]`, the RHS of `=` is meaningful.** Quoted = literal
  compare. Unquoted = glob match. Reviewers should flag mismatches
  between intent and form. Source:
  [BashPitfalls/34](https://mywiki.wooledge.org/BashPitfalls#pf34),
  [SC2053](https://www.shellcheck.net/wiki/SC2053).
- **Inside `[[ ]]`, the RHS of `=~` should NOT be quoted.** Quoted
  regex becomes a literal string match. Store the pattern in a
  variable if it's complex:
  ```bash
  pat='^[0-9]+$'
  [[ $x =~ $pat ]]
  ```
  Source: [SC2076](https://www.shellcheck.net/wiki/SC2076),
  [BashPitfalls/35](https://mywiki.wooledge.org/BashPitfalls#pf35).
- **`[ ]` requires quoting; `[[ ]]` mostly doesn't.** With `[ -n $x ]`,
  if `$x` is empty the command becomes `[ -n ]`, which is true (because
  `-n` with no argument is true). Always quote inside `[ ]`. Source:
  [SC2070](https://www.shellcheck.net/wiki/SC2070),
  [BashPitfalls/36](https://mywiki.wooledge.org/BashPitfalls#pf36).
- **Tilde does not expand inside quotes.** `"~/file"` is literal;
  `"$HOME/file"` is what you want. Source:
  [BashPitfalls/26](https://mywiki.wooledge.org/BashPitfalls#pf26),
  [SC2088](https://www.shellcheck.net/wiki/SC2088).

## Test commands and conditionals

- **`[[ ]]` over `[ ]` and `test`** in bash. `[[` is a keyword: no
  word splitting, no pathname expansion of operands, supports `=~`
  regex and `==` glob. `[` is a regular command with POSIX-shaped
  gotchas. Source:
  [BashFAQ/031](https://mywiki.wooledge.org/BashFAQ/031),
  [Google s6.3](https://google.github.io/styleguide/shellguide.html#s6.3-tests).
- **`(( ))` for numeric comparison.** `<` and `>` inside `[[ ]]` are
  string comparisons, not numeric. `[[ 10 > 9 ]]` is false (because
  "10" < "9" lexically). Use `(( 10 > 9 ))` or `[[ 10 -gt 9 ]]`.
  Source: [BashPitfalls/7](https://mywiki.wooledge.org/BashPitfalls#pf7),
  [SC2071](https://www.shellcheck.net/wiki/SC2071),
  [Google s6.4](https://google.github.io/styleguide/shellguide.html#s6.4-testing-strings).
- **`-z`/`-n` over `[ "$x" = "" ]`.** Explicit zero/non-zero string
  tests. Source: Google s6.4.
- **Test exit codes directly.** `if cmd; then …` over
  `cmd; if (( $? == 0 )); then …`. Capturing `$?` separately invites
  it being overwritten by an intervening command, and breaks `set -e`.
  Source: [SC2181](https://www.shellcheck.net/wiki/SC2181).
- **`if [[ p && q ]]`** inside `[[ ]]`, not `-a` (which is undefined
  for multiple operands). `[ p -a q ]` is technically POSIX but the
  semantics are ambiguous; split into `[ p ] && [ q ]`. Source:
  [SC2166](https://www.shellcheck.net/wiki/SC2166),
  [SC2107](https://www.shellcheck.net/wiki/SC2107).
- **`A && B || C` is not if-then-else.** If `B` fails, `C` runs even
  though `A` succeeded. Use `if A; then B; else C; fi`. Source:
  [BashPitfalls/22](https://mywiki.wooledge.org/BashPitfalls#pf22),
  [SC2015](https://www.shellcheck.net/wiki/SC2015).

## Subshells and pipelines

- **Variables set in a piped loop don't survive.** Each pipe stage runs
  in a subshell. The classic `cmd | while read x; do ((n++)); done`
  loses `n` after the loop. Source:
  [BashFAQ/024](https://mywiki.wooledge.org/BashFAQ/024),
  [BashPitfalls/8](https://mywiki.wooledge.org/BashPitfalls#pf8),
  [SC2030](https://www.shellcheck.net/wiki/SC2030)/[SC2031](https://www.shellcheck.net/wiki/SC2031).
- **Fixes:**
  - Process substitution: `while read x; do …; done < <(cmd)`.
  - `mapfile -t arr < <(cmd)`, then loop over `arr`.
  - `shopt -s lastpipe` (bash 4.2+, requires `set +m`).
  - Here-string from a captured value: `while … done <<< "$captured"`.
- **`(…)` is a subshell; `{ …; }` is not.** Reviewers should flag
  subshells used where grouping was wanted, because variable changes
  inside `(…)` are lost. Source:
  [SC2235](https://www.shellcheck.net/wiki/SC2235).
- **`local var=$(cmd)` masks `cmd`'s exit status.** `local` itself
  returns 0. Split: `local var; var=$(cmd)`. Source:
  [SC2155](https://www.shellcheck.net/wiki/SC2155),
  [BashFAQ/105](https://mywiki.wooledge.org/BashFAQ/105) (one of
  set -e's blind spots).
- **`while … done <<< "$(cmd)"`** collects all output first into a
  here-string. For streaming, use process substitution. Source:
  [BashPitfalls/63](https://mywiki.wooledge.org/BashPitfalls#pf63).

## Reading input

- **`read -r` always.** Without `-r`, backslashes are interpreted; you
  almost never want this. Source:
  [SC2162](https://www.shellcheck.net/wiki/SC2162).
- **`IFS= read -r line`** for whole lines. `IFS=` prevents
  leading/trailing whitespace trim. Source:
  [BashFAQ/001](https://mywiki.wooledge.org/BashFAQ/001).
- **Handle files without trailing newline:**
  ```bash
  while IFS= read -r line || [[ -n $line ]]; do
    …
  done < "$file"
  ```
  BashFAQ/001.
- **`mapfile -t arr < "$file"`** is faster and clearer than the
  append-in-a-loop pattern.
- **Don't iterate `ls`.** Filenames may contain newlines or globs that
  word-splitting interprets. Use globs (`for f in *.txt`) or
  `find -print0 | while IFS= read -r -d '' f`. Source:
  [SC2045](https://www.shellcheck.net/wiki/SC2045),
  [BashPitfalls/1](https://mywiki.wooledge.org/BashPitfalls#pf1).
- **Don't iterate `for line in $(cat file)`.** Word splitting + glob
  expansion; use `while IFS= read -r line; do …; done < file`. Source:
  [SC2013](https://www.shellcheck.net/wiki/SC2013).
- **`ssh` consumes stdin in a read loop.** Common surprise: a
  `while read line; do ssh host …; done < file` processes only the
  first line because ssh swallows the rest. Fixes: `ssh -n`, redirect
  ssh's stdin: `< /dev/null`, or use file descriptor isolation
  (`done 9< file` with `read -r line <&9`). Source:
  [BashFAQ/089](https://mywiki.wooledge.org/BashFAQ/089),
  [SC2095](https://www.shellcheck.net/wiki/SC2095).

## Substitutions: `$()` and `printf`

- **`$(cmd)` over backticks.** Backticks have weird backslash rules
  and nest unreadably. Source:
  [BashFAQ/082](https://mywiki.wooledge.org/BashFAQ/082),
  [SC2006](https://www.shellcheck.net/wiki/SC2006),
  [Google s6.2](https://google.github.io/styleguide/shellguide.html#s6.2-command-substitution).
- **`$(<file)` over `$(cat file)`.** Builtin form, no fork. (Note:
  both strip trailing newlines, per
  [BashPitfalls/41](https://mywiki.wooledge.org/BashPitfalls#pf41).)
- **`printf '%s\n' "$x"` over `echo "$x"`.** `echo` behaviour for
  arguments starting with `-`, containing `\`, or in `xpg_echo` mode
  is unspecified. `printf` is portable and unambiguous. Source:
  [SC2028](https://www.shellcheck.net/wiki/SC2028).
- **Never put a variable in the printf format string.** `printf "$x"`
  is a format-string vulnerability: if `$x` contains `%`, it triggers
  format interpretation; if `$x` is hostile, it can crash or worse.
  Always `printf '%s\n' "$x"` or `printf '%s' "$x"`. Source:
  [BashPitfalls/32](https://mywiki.wooledge.org/BashPitfalls#pf32),
  [SC2059](https://www.shellcheck.net/wiki/SC2059).
- **`printf -v var FORMAT ARGS`** assigns to a variable without
  forking a subshell. Use instead of `var=$(printf …)`.

## Arithmetic

- **`$(( ))` for integer math; never `expr`.** Source:
  [Google s6.9](https://google.github.io/styleguide/shellguide.html#s6.9-arithmetic),
  [SC2003](https://www.shellcheck.net/wiki/SC2003).
- **Leading-zero values are octal in `$(( ))`.** `$(( 010 ))` is 8.
  Force decimal: `$(( 10#$n ))`. Source:
  [SC2080](https://www.shellcheck.net/wiki/SC2080),
  [BashPitfalls/59](https://mywiki.wooledge.org/BashPitfalls#pf59).
- **No floats in bash.** Use `bc`, `awk`, or push the math into `jq`
  or Python. Source:
  [BashWeaknesses](https://mywiki.wooledge.org/BashWeaknesses).
- **Arithmetic context evaluates strings as code.** `(( a[$key]++ ))`
  with hostile `$key` allows code injection. Validate keys first.
  Source: [BashPitfalls/62](https://mywiki.wooledge.org/BashPitfalls#pf62).
- **`let` is deprecated;** prefer `(( ))`. Source:
  [SC2219](https://www.shellcheck.net/wiki/SC2219).
- **`(( expr ))` returns non-zero if `expr` is zero.** Under `set -e`
  a standalone `(( count = 0 ))` aborts the script. Source: Google
  s6.9.

## `cd` safety

- **`cd "$dir" || exit`** after every `cd`. (Or `|| return` inside a
  function.) A silent `cd` failure leaves subsequent commands running
  in the wrong directory; with `rm -rf` downstream this is
  destructive. Source:
  [SC2164](https://www.shellcheck.net/wiki/SC2164),
  [BashPitfalls/19](https://mywiki.wooledge.org/BashPitfalls#pf19).
- **Or use `(cd "$dir" && …)`** in a subshell so the parent doesn't
  need to `cd` back.

## Error handling: the strict-mode tension

See [strict-mode.md](strict-mode.md) for the full debate, the seven
documented blind spots with examples, opt-in shellcheck checks
(`check-set-e-suppressed`, `check-extra-masked-returns`), and the
reviewer heuristic.

In short: Google and Aaron Maxwell recommend `set -euo pipefail;
IFS=$'\n\t'`. Wooledge documents the silent blind spots
([BashFAQ/105](https://mywiki.wooledge.org/BashFAQ/105),
[BashFAQ/112](https://mywiki.wooledge.org/BashFAQ/112),
[BashPitfalls/60](https://mywiki.wooledge.org/BashPitfalls#pf60))
and recommends explicit `cmd || { msg; exit 1; }`. Report both;
don't insert strict mode reflexively; don't strip it from a script
that has it.

Mechanical correctness checks that apply regardless of strict-mode
choice:

- `cd "$dir" || exit` after every `cd`
  ([SC2164](https://www.shellcheck.net/wiki/SC2164)).
- `local x; x=$(cmd)` not `local x=$(cmd)`
  ([SC2155](https://www.shellcheck.net/wiki/SC2155)).
- `if cmd; then …` not `cmd; if (( $? == 0 )); then`
  ([SC2181](https://www.shellcheck.net/wiki/SC2181)).

## eval and code-as-data

- **Avoid `eval`.** Source:
  [BashFAQ/048](https://mywiki.wooledge.org/BashFAQ/048),
  [Google s6.6](https://google.github.io/styleguide/shellguide.html#s6.6-eval).
- **"Variables hold data; functions hold code."** Storing commands in
  strings invites bugs and injection. Use arrays for argument lists,
  functions for reusable command sequences. Source:
  [BashFAQ/050](https://mywiki.wooledge.org/BashFAQ/050) (canonical).
- **The array pattern:**
  ```bash
  args=(-v --output "$out")
  [[ $verbose ]] && args+=(--verbose)
  cmd "${args[@]}"
  ```
- **Parameter expansion for conditional flags:**
  ```bash
  cmd ${count:+"-c$count"} -- "$target"
  ```
- **Indirect variables:** prefer `declare -n ref="$name"` (bash 4.3+)
  over `eval "echo \$$name"`. (Note: name references also have
  injection risks; validate the name.) Source:
  [BashFAQ/006](https://mywiki.wooledge.org/BashFAQ/006).

## Race conditions and temp files

- **`mktemp` over `$$` filenames.** `/tmp/script.$$` is predictable
  and racy. `mktemp` returns a guaranteed-unique name with the right
  permissions. Source:
  [BashFAQ/062](https://mywiki.wooledge.org/BashFAQ/062),
  [SC2186](https://www.shellcheck.net/wiki/SC2186).
- **`trap 'rm -rf "$tmpdir"' EXIT`** to clean up on any exit.
- **`flock` for instance locking,** not pidfiles. Source:
  [BashFAQ/045](https://mywiki.wooledge.org/BashFAQ/045).
- **Never read and write the same file in a pipeline.**
  `cat f | sed s/x/y/ > f` truncates `f` before sed reads it. Use
  `sed -i` or write to a tmp file then `mv`. Source:
  [BashPitfalls/13](https://mywiki.wooledge.org/BashPitfalls#pf13),
  [SC2094](https://www.shellcheck.net/wiki/SC2094).
- **`sudo cmd > /file`** runs the redirection as the calling user.
  Wrap: `sudo sh -c 'cmd > /file'` or `cmd | sudo tee /file`. Source:
  [BashPitfalls/53](https://mywiki.wooledge.org/BashPitfalls#pf53),
  [SC2024](https://www.shellcheck.net/wiki/SC2024).

## Signal handling and traps

- **`trap '…' EXIT`** for cleanup that should run on any exit.
- **Trap by name, not number.** `trap … INT TERM` not `trap … 2 15`;
  signal numbers vary across platforms. Source:
  [SC2172](https://www.shellcheck.net/wiki/SC2172).
- **`trap` argument should be single-quoted** so expansions happen
  when the trap fires, not when it's set. Source:
  [SC2064](https://www.shellcheck.net/wiki/SC2064).
- **Don't trap SIGKILL or SIGSTOP** (can't be caught). Source:
  [SC2173](https://www.shellcheck.net/wiki/SC2173).

## Redirections

- **Order matters: `cmd >file 2>&1`** sends both to `file`.
  `cmd 2>&1 >file` sends stderr to wherever stdout was (terminal),
  then redirects stdout to file. Source:
  [BashPitfalls/43](https://mywiki.wooledge.org/BashPitfalls#pf43),
  [SC2069](https://www.shellcheck.net/wiki/SC2069).
- **Close vs redirect:** `2>&-` closes stderr; some programs choke on
  that. `2>/dev/null` discards. Prefer the latter. Source:
  [BashPitfalls/55](https://mywiki.wooledge.org/BashPitfalls#pf55).

## Related

- [pitfalls.md](pitfalls.md): the full BashPitfalls 1-65 catalogue.
- [shellcheck-codes.md](shellcheck-codes.md): the SC code lookup.
- [idioms.md](idioms.md): the fluent patterns.
