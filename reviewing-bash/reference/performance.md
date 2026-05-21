# Bash performance reference

Bash is slow ([BashWeaknesses](https://mywiki.wooledge.org/BashWeaknesses)).
For scripts that run rarely the cost rarely matters. For statuslines,
prompts, git hooks, completions, or anything in a tight loop, every
fork compounds.

## Contents

- The fork cost
- Builtins beat externals
- Single-pass tools
- Hot-loop hoisting
- Useless-use-of patterns
- When to leave bash

## The fork cost

Every external command incurs `fork()` plus `execve()`. On Linux this
is ~1 ms; on macOS it is ~3-5 ms. A loop of 1000 iterations each
spawning one external is a second of pure overhead. Builtins (the
commands listed under `help` inside bash) run in-process and cost
microseconds.

`$(cmd)` and pipes also spawn subshells, which is a cheaper fork (no
exec) but not free. `printf -v` and parameter expansion don't fork at
all.

## Builtins beat externals

Prefer the left column over the right.

| Use | Instead of |
|-----|------------|
| `[[ -f f ]]`, `[[ -d d ]]` | `test -f f`, `/usr/bin/[ -d d ]` |
| `${var//x/y}` | `echo "$var" \| sed s/x/y/g` |
| `${var,,}` (lowercase) | `echo "$var" \| tr A-Z a-z` |
| `${#var}` (length) | `echo "$var" \| wc -c` |
| `${var##*/}` | `basename "$var"` |
| `${var%/*}` | `dirname "$var"` |
| `printf -v var FMT …` | `var=$(printf FMT …)` |
| `$(< file)` | `$(cat file)` |
| `read -r var < file` | `var=$(head -n1 file)` |
| `(( a < b ))` | `[ "$a" -lt "$b" ]` (test is external on some systems) |
| `command -v cmd` | `which cmd` ([SC2230](https://www.shellcheck.net/wiki/SC2230)) |
| `mapfile -t arr < file` | manual append-in-a-loop |

Source:
[Google s8.2](https://google.github.io/styleguide/shellguide.html#s8.2-builtin-commands-vs-external-commands),
[BashGuide/Practices](https://mywiki.wooledge.org/BashGuide/Practices),
multiple
[SC2001](https://www.shellcheck.net/wiki/SC2001),
[SC2002](https://www.shellcheck.net/wiki/SC2002),
[SC2005](https://www.shellcheck.net/wiki/SC2005),
[SC2126](https://www.shellcheck.net/wiki/SC2126).

## Single-pass tools

One invocation of `jq`/`awk`/`sed` processing N records beats N
invocations processing one record each. The startup time of the
interpreter dominates for trivial scripts.

```bash
# Slow: N spawns of jq
for id in "${ids[@]}"; do
  name=$(jq -r ".items[\"$id\"].name" data.json)
  echo "$name"
done

# Fast: one spawn of jq, processes all
jq -r --argjson ids "$(printf '%s\n' "${ids[@]}" | jq -R . | jq -s .)" \
  '.items[$ids[]] | .name' data.json
```

```bash
# Slow: N sed spawns
for f in *.conf; do
  sed -i 's/old/new/' "$f"
done

# Faster: one sed spawn over many files
sed -i 's/old/new/' *.conf
```

`awk` is particularly good at this: any per-line work that needs more
than a substitution belongs in one `awk` call.

## Hot-loop hoisting

Move work out of loops when the result doesn't change per iteration.

```bash
# Bad: spawns date every iteration
for f in *.log; do
  ts=$(date +%s)
  echo "$ts $f" >> processed
done

# Good: spawn once
ts=$(date +%s)
for f in *.log; do
  echo "$ts $f" >> processed
done
```

```bash
# Bad: spawns grep per file
for f in *.txt; do
  if grep -q ERROR "$f"; then keep+=("$f"); fi
done

# Better: one grep across all files
mapfile -t keep < <(grep -l ERROR ./*.txt)
```

## Useless-use-of patterns

Common patterns flagged by shellcheck that also signal fork waste:

- **UUOC (`cat file | cmd`):** the pipe and cat are both wasted. Use
  `cmd < file` or `cmd file` if `cmd` reads filenames.
  [SC2002](https://www.shellcheck.net/wiki/SC2002) catches it but is
  shellcheck opt-in (`-o useless-use-of-cat`). Flag it in review
  whether or not the opt-in is enabled.
- **`echo $(cmd)`:** the `$()` and `echo` together strip newlines and
  add one back. If you want the output, just run `cmd`. Source:
  [SC2005](https://www.shellcheck.net/wiki/SC2005).
- **`grep pattern file | wc -l`:** use `grep -c pattern file`. Source:
  [SC2126](https://www.shellcheck.net/wiki/SC2126).
- **`ls | grep`:** use a glob, `find`, or `compgen`. Source:
  [SC2010](https://www.shellcheck.net/wiki/SC2010).
- **`sed 's/old/new/g'` for trivial replace:** parameter expansion
  `${var//old/new}`. Source:
  [SC2001](https://www.shellcheck.net/wiki/SC2001).
- **`echo "$var" | sed`:** combine into `<<<`: `sed … <<< "$var"`,
  or use parameter expansion entirely.

## Caching expensive calls

For repeated lookups inside a script, cache the answer:

```bash
declare -A _cache
lookup() {
  local key=$1
  if [[ -z ${_cache[$key]+set} ]]; then
    _cache[$key]=$(expensive_command "$key")
  fi
  printf '%s\n' "${_cache[$key]}"
}
```

The `${var+set}` form distinguishes "set but empty" from "unset".

## When to leave bash

Performance signals that mean bash is the wrong language:

- Inner loop has > 1 ms ceiling (statusline target, hot completion).
- Doing string parsing more complex than `${//}` per item.
- Needing concurrent I/O across many endpoints
  ([BashWeaknesses](https://mywiki.wooledge.org/BashWeaknesses) on
  process management).
- Needing data structures beyond flat arrays and associative arrays.
- Crossing 100 lines or non-linear control flow. Google
  [s1.2](https://google.github.io/styleguide/shellguide.html#s1.2-when-to-use-shell)
  is explicit: rewrite in a structured language at that point.

See [when-not-to-use-bash.md](when-not-to-use-bash.md) for the full
signal set.

## Measurement

When in doubt, measure. `time` works on pipelines and compounds with
`bash -c`:

```bash
time bash -c 'for i in {1..1000}; do echo "$i" | sed s/1/X/; done >/dev/null'
time bash -c 'for i in {1..1000}; do printf "%s\n" "${i//1/X}"; done >/dev/null'
```

Often the difference is one order of magnitude.

Use `bash -x` to see expansion in real time when investigating "why
is this slow" with branching scripts.

## Related

- [idioms.md](idioms.md): the patterns that avoid forks.
- [correctness.md](correctness.md#substitutions-and-printf): `$()`
  and `printf` correctness.
