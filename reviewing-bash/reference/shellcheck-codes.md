# ShellCheck codes reference

The SC codes most-encountered in review, with cause and fix. For the
full catalogue, see
[the wiki](https://www.shellcheck.net/wiki/) or
[the gh wiki Checks page](https://github.com/koalaman/shellcheck/wiki/Checks).

## Contents

- Code prefix meanings
- Severity tiers
- Default vs opt-in checks
- Quoting and expansion
- Tests and conditions
- Reading and looping
- Substitutions and external commands
- Arithmetic and integer compare
- Functions, variables, scope
- Find, xargs, ls
- Subshells, pipelines, exit codes
- Redirection and sudo
- printf and echo
- Style and portability
- Opt-in checks worth enabling
- Disabling a check

## Code prefix meanings

| Prefix | Meaning |
|--------|---------|
| SC1xxx | Parser errors and syntax issues (script won't run) |
| SC2xxx | Runtime issues, logic errors, best-practice violations |
| SC3xxx | POSIX shell compatibility (only when target is POSIX `sh`) |

Most review findings sit in SC2xxx.

## Severity tiers

ShellCheck assigns each finding a severity:

- **error**: almost certainly broken. Treat as a hard fail.
- **warning**: likely broken or definitely fragile. Investigate.
- **info**: stylistic or possible-improvement.
- **style**: cosmetic, lowest priority.

`shellcheck -S warning <path>` filters out info/style. Use full
output for thorough review; filter for hot paths.

## Default vs opt-in checks

Some shellcheck checks are **opt-in** and silent by default. They
fire only when explicitly enabled with `-o name` or `-o all`. The
most relevant for review:

- SC2002 (useless cat) requires `-o useless-use-of-cat`.
- SC2312 (command-substitution exit-code masking under `set -e`)
  requires `-o check-extra-masked-returns`.
- Some POSIX/style checks like SC2112 (`function` keyword) fire
  only under `-s sh` dialect, not `-s bash`, because bash accepts
  the construct.

Reviewers should know which findings their default `shellcheck` run
will miss. The full opt-in list is below; run
`shellcheck --list-optional` for descriptions and examples.

## Quoting and expansion

| Code | Title | Fix |
|------|-------|-----|
| [SC2086](https://www.shellcheck.net/wiki/SC2086) | Double quote to prevent globbing and word splitting | `"$var"` instead of `$var` |
| [SC2068](https://www.shellcheck.net/wiki/SC2068) | Double quote array expansions to avoid re-splitting | `"${arr[@]}"` |
| [SC2048](https://www.shellcheck.net/wiki/SC2048) | Use `"$@"` (with quotes) | `"$@"` not `$@` or `$*` |
| [SC2046](https://www.shellcheck.net/wiki/SC2046) | Quote this to prevent word splitting | `"$(cmd)"` |
| [SC2053](https://www.shellcheck.net/wiki/SC2053) | Quote the RHS of `=` in `[[ ]]` for literal match | `[[ $a = "$b" ]]` |
| [SC2076](https://www.shellcheck.net/wiki/SC2076) | Don't quote RHS of `=~`; will match literally | unquote regex |
| [SC2088](https://www.shellcheck.net/wiki/SC2088) | Tilde does not expand in quotes | `"$HOME/..."` |
| [SC2089](https://www.shellcheck.net/wiki/SC2089) / [SC2090](https://www.shellcheck.net/wiki/SC2090) | Quotes/backslashes in this variable will not be respected | Use an array |
| [SC2027](https://www.shellcheck.net/wiki/SC2027) | The surrounding quotes actually unquote this | Restructure |
| [SC2070](https://www.shellcheck.net/wiki/SC2070) | `-n` doesn't work with unquoted arguments | Quote, or `[[ ]]` |

## Tests and conditions

| Code | Title | Fix |
|------|-------|-----|
| [SC2071](https://www.shellcheck.net/wiki/SC2071) | `>` is for string comparisons. Use `-gt` | `(( a > b ))` or `[[ a -gt b ]]` |
| [SC2166](https://www.shellcheck.net/wiki/SC2166) | Prefer `[ p ] && [ q ]` as `[ p -a q ]` is not well-defined | Split into two tests |
| [SC2107](https://www.shellcheck.net/wiki/SC2107) | Instead of `[ a && b ]`, use `[ a ] && [ b ]` | Split |
| [SC2236](https://www.shellcheck.net/wiki/SC2236) | Use `-n` instead of `! -z` | `[ -n "$x" ]` |
| [SC2199](https://www.shellcheck.net/wiki/SC2199) | Arrays implicitly concatenate in `[[ ]]` | Use a loop |
| [SC2049](https://www.shellcheck.net/wiki/SC2049) | `=~` is for regex, but this looks like a glob | Use `==` for glob |
| [SC2050](https://www.shellcheck.net/wiki/SC2050) | This expression is constant. Forgot `$`? | Add `$` |
| [SC2143](https://www.shellcheck.net/wiki/SC2143) | Use `grep -q` instead of `[ -n "$(... \| grep)" ]` | `if grep -q …; then` |

## Reading and looping

| Code | Title | Fix |
|------|-------|-----|
| [SC2162](https://www.shellcheck.net/wiki/SC2162) | `read` without `-r` mangles backslashes | `read -r` |
| [SC2013](https://www.shellcheck.net/wiki/SC2013) | To read lines rather than words, pipe to `while read` | `while IFS= read -r …; do` |
| [SC2044](https://www.shellcheck.net/wiki/SC2044) | For loops over `find` output are fragile | `find … -exec` or `while read -d ''` |
| [SC2045](https://www.shellcheck.net/wiki/SC2045) | Iterating over `ls` output is fragile | Use globs or `find` |
| [SC2066](https://www.shellcheck.net/wiki/SC2066) | Since you double-quoted, the loop runs once | Unquote intentionally, or rethink |
| [SC2231](https://www.shellcheck.net/wiki/SC2231) | Quote expansions in for-loop glob | `for f in "$dir"/*.txt` |
| [SC2043](https://www.shellcheck.net/wiki/SC2043) | This loop will only ever run once | Did you mean `*` or `$var`? |

## Substitutions and external commands

| Code | Title | Fix |
|------|-------|-----|
| [SC2006](https://www.shellcheck.net/wiki/SC2006) | Use `$(...)` not legacy backticks | `$(cmd)` |
| [SC2002](https://www.shellcheck.net/wiki/SC2002) **opt-in** | Useless cat. `cmd < file` | Drop the cat. Enable with `-o useless-use-of-cat` |
| [SC2005](https://www.shellcheck.net/wiki/SC2005) | Useless `echo`? `echo $(cmd)` → just `cmd` | Drop the echo |
| [SC2126](https://www.shellcheck.net/wiki/SC2126) | Consider `grep -c` instead of `grep \| wc -l` | `grep -c` |
| [SC2001](https://www.shellcheck.net/wiki/SC2001) | Use `${var//search/replace}` instead of `sed` | parameter expansion |
| [SC2003](https://www.shellcheck.net/wiki/SC2003) | `expr` is antiquated; use `$(( ))` | `$(( a + b ))` |
| [SC2230](https://www.shellcheck.net/wiki/SC2230) | `which` is non-standard; use `command -v` | `command -v cmd` |

## Arithmetic and integer compare

| Code | Title | Fix |
|------|-------|-----|
| [SC2004](https://www.shellcheck.net/wiki/SC2004) | `$/${}` is unnecessary on arithmetic variables | `(( a + b ))` not `(( $a + $b ))` |
| [SC2007](https://www.shellcheck.net/wiki/SC2007) | Use `$(( ))` instead of deprecated `$[ … ]` | `$(( ))` |
| [SC2080](https://www.shellcheck.net/wiki/SC2080) | Numbers with leading 0 are octal | `$(( 10#$n ))` |
| [SC2219](https://www.shellcheck.net/wiki/SC2219) | Prefer `(( expr ))` over `let expr` | `(( i++ ))` |
| [SC2099](https://www.shellcheck.net/wiki/SC2099) / [SC2100](https://www.shellcheck.net/wiki/SC2100) | Use `$(( ))` for arithmetic | `i=$((i + 2))` |

## Functions, variables, scope

| Code | Title | Fix |
|------|-------|-----|
| [SC2155](https://www.shellcheck.net/wiki/SC2155) | Declare and assign separately to avoid masking return values | `local x; x=$(cmd)` |
| [SC2034](https://www.shellcheck.net/wiki/SC2034) | foo appears unused | Remove, or `export`, or comment why kept |
| [SC2154](https://www.shellcheck.net/wiki/SC2154) | `var` is referenced but not assigned | Typo? Or initialize first |
| [SC2153](https://www.shellcheck.net/wiki/SC2153) | Possible misspelling: MYVAR may not be assigned, but MY_VAR is | Check spelling |
| [SC2168](https://www.shellcheck.net/wiki/SC2168) | `local` is only valid in functions | Remove or move into function |
| [SC2178](https://www.shellcheck.net/wiki/SC2178) | Variable was used as array but is now assigned a string | Use `arr+=("x")` |
| [SC2128](https://www.shellcheck.net/wiki/SC2128) | Expanding array without index gives first element | Use `"${arr[@]}"` |
| [SC2206](https://www.shellcheck.net/wiki/SC2206) / [SC2207](https://www.shellcheck.net/wiki/SC2207) | Prefer `mapfile` or `read -a` to split | `mapfile -t arr < <(cmd)` |
| [SC2123](https://www.shellcheck.net/wiki/SC2123) | `PATH` is the shell search path. Use another name | Rename |
| [SC2120](https://www.shellcheck.net/wiki/SC2120) / [SC2119](https://www.shellcheck.net/wiki/SC2119) | Function references args, but none are passed | Pass `"$@"` |

## Find, xargs, ls

| Code | Title | Fix |
|------|-------|-----|
| [SC2010](https://www.shellcheck.net/wiki/SC2010) | Don't use `ls \| grep` | Use glob or `for` loop |
| [SC2011](https://www.shellcheck.net/wiki/SC2011) | Use `find -print0`/`-exec` for non-alphanumeric names | `find … -exec … +` |
| [SC2012](https://www.shellcheck.net/wiki/SC2012) | Use `find` instead of `ls` for non-alphanumeric names | `find` |
| [SC2038](https://www.shellcheck.net/wiki/SC2038) | Use `-print0`/`-0` for non-alphanumeric filenames | NUL-delim |
| [SC2156](https://www.shellcheck.net/wiki/SC2156) | Injecting filenames is fragile | Use `find -exec` args form |
| [SC2150](https://www.shellcheck.net/wiki/SC2150) | `-exec` does not invoke a shell | `find … -exec sh -c '…' _ {} \;` |

## Subshells, pipelines, exit codes

| Code | Title | Fix |
|------|-------|-----|
| [SC2030](https://www.shellcheck.net/wiki/SC2030) | Modification of var is local (subshell) | Use process substitution |
| [SC2031](https://www.shellcheck.net/wiki/SC2031) | var was modified in a subshell. Change might be lost | Same fix |
| [SC2181](https://www.shellcheck.net/wiki/SC2181) | Check exit code directly: `if cmd;` not `if [ $? = … ]` | `if cmd; then …` |
| [SC2015](https://www.shellcheck.net/wiki/SC2015) | `A && B \|\| C` is not if-then-else | Use real `if`/`else` |
| [SC2164](https://www.shellcheck.net/wiki/SC2164) | Use `cd ... \|\| exit` in case cd fails | `cd "$d" \|\| exit` |
| [SC2103](https://www.shellcheck.net/wiki/SC2103) | Use `( subshell )` to avoid cd back | `(cd dir && cmd)` |
| [SC2106](https://www.shellcheck.net/wiki/SC2106) | This only exits the subshell caused by the pipeline | Restructure |
| [SC2235](https://www.shellcheck.net/wiki/SC2235) | Use `{ ..; }` instead of `(..)` to avoid subshell | `{ cmd1; cmd2; }` |

## Redirection and sudo

| Code | Title | Fix |
|------|-------|-----|
| [SC2024](https://www.shellcheck.net/wiki/SC2024) | `sudo` doesn't affect redirects | `sudo tee` or `sudo sh -c` |
| [SC2069](https://www.shellcheck.net/wiki/SC2069) | To redirect stdout+stderr, `2>&1` must be last | `>file 2>&1` |
| [SC2094](https://www.shellcheck.net/wiki/SC2094) | Don't read and write the same file in the same pipeline | tmp file + mv |
| [SC2129](https://www.shellcheck.net/wiki/SC2129) | Consider `{ cmd1; cmd2; } >> file` | Group redirects |

## printf and echo

| Code | Title | Fix |
|------|-------|-----|
| [SC2028](https://www.shellcheck.net/wiki/SC2028) | `echo` won't expand escape sequences. Consider `printf` | `printf '%s\n' "$x"` |
| [SC2059](https://www.shellcheck.net/wiki/SC2059) | Don't use variables in the printf format string | `printf '%s\n' "$x"` |
| [SC2008](https://www.shellcheck.net/wiki/SC2008) | echo doesn't read from stdin | Use `cat <<EOF` |

## Style and portability

| Code | Title | Fix |
|------|-------|-----|
| [SC1090](https://www.shellcheck.net/wiki/SC1090) | Can't follow non-constant source | `# shellcheck source=path` |
| [SC1091](https://www.shellcheck.net/wiki/SC1091) | Not following sourced file | `# shellcheck source=path` |
| [SC2148](https://www.shellcheck.net/wiki/SC2148) | Add a shebang line | `#!/usr/bin/env bash` |
| [SC2112](https://www.shellcheck.net/wiki/SC2112) (POSIX only) | `function` keyword is non-standard | `name() { ... }`. Fires only under `-s sh`; bash accepts it |
| [SC2184](https://www.shellcheck.net/wiki/SC2184) | Quote arguments to unset to prevent glob | `unset 'arr[0]'` |
| [SC2064](https://www.shellcheck.net/wiki/SC2064) | Use single quotes in trap, otherwise expansion happens now | `trap 'rm "$f"' EXIT` |
| [SC2172](https://www.shellcheck.net/wiki/SC2172) | Trapping signals by number is not well-defined | `trap … INT TERM` |
| [SC2173](https://www.shellcheck.net/wiki/SC2173) | SIGKILL/SIGSTOP can not be trapped | Remove |
| [SC2186](https://www.shellcheck.net/wiki/SC2186) | tempfile is deprecated. Use mktemp | `mktemp` |
| [SC1082](https://www.shellcheck.net/wiki/SC1082) | This file has a UTF-8 BOM. Remove it | `LC_CTYPE=C sed '1s/^...//'` |
| [SC1128](https://www.shellcheck.net/wiki/SC1128) | Shebang must be on the first line | Move shebang up |

## Opt-in checks worth enabling

Run with `shellcheck -o name` (or `-o all` to enable everything;
verify nothing breaks before using in CI).

| Opt-in | What it catches |
|--------|-----------------|
| `useless-use-of-cat` | SC2002. `cat file \| cmd` |
| `check-set-e-suppressed` | functions called from `if`/`&&`/`\|\|` silently disable `set -e` inside |
| `check-extra-masked-returns` | command-substitution exit-code masking under `set -e` (SC2312) |
| `quote-safe-variables` | unquoted vars even when currently safe (defends against later edits making them unsafe) |
| `require-double-brackets` | flags `[ ]` in bash/ksh, suggests `[[ ]]` |
| `require-variable-braces` | flags `$var` without `${var}` |
| `deprecate-which` | suggests `command -v` over `which` |
| `check-unassigned-uppercase` | uppercase variable names that look exported but aren't |
| `add-default-case` | suggests adding `*)` to `case` statements |
| `avoid-nullary-conditions` | suggests `[ -n "$var" ]` over `[ "$var" ]` |
| `avoid-negated-conditions` | suggests `[ "$x" -ne 1 ]` over `[ ! "$x" -eq 1 ]` |

`check-set-e-suppressed` is the most useful for strict-mode review;
see [strict-mode.md](strict-mode.md).

## Disabling a check

Inline disable for one line:

```bash
# shellcheck disable=SC2086  # intentional word splitting
read -ra parts <<< $LINE
```

File-level disable at the top:

```bash
#!/usr/bin/env bash
# shellcheck disable=SC2034
```

External directives for sourced files:

```bash
# shellcheck source=./lib/util.sh
source "${LIB_DIR}/util.sh"
```

Disables should always have a one-line justification next to them.
A bare `# shellcheck disable=…` with no reason is a code smell;
flag those during review.

## Related

- [correctness.md](correctness.md): the principles behind these codes.
- [pitfalls.md](pitfalls.md): the BashPitfalls numbering, which
  overlaps but is organized differently.
