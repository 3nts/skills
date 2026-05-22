# Strict mode: the unresolved debate

`set -euo pipefail` plus `IFS=$'\n\t'` is the "Unofficial Bash Strict
Mode" preamble, popularized by Aaron Maxwell and recommended by
Google's style guide. Wooledge documents its blind spots in
BashFAQ/105, BashFAQ/112, and BashPitfalls/60 and recommends against
it. There is no consensus.

Reviewers should report both positions; not insert strict mode
reflexively into a script that doesn't use it; not strip it from a
script that does.

## Contents

- The preamble
- The pro-strict position
- The anti-strict position
- The blind spots, with examples
- Opt-in shellcheck checks for strict mode
- Migration patterns
- Reviewer heuristic

## The preamble

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
```

- `set -e` (`errexit`): exit on first command failure.
- `set -u` (`nounset`): unset variables abort instead of expanding to
  empty.
- `set -o pipefail`: pipeline exit non-zero if any stage fails (not
  just the last).
- `IFS=$'\n\t'`: word splitting only on tab/newline.

Source: Aaron Maxwell,
[Unofficial Bash Strict Mode](http://redsymbol.net/articles/unofficial-bash-strict-mode/);
Google
[s1.1](https://google.github.io/styleguide/shellguide.html#s1.1-which-shell-to-use).

## The pro-strict position

The argument:

- Without `set -e`, a silent failure in line 3 leaves lines 4-100
  running on bad data. Most scripts don't want that, even though most
  scripts don't notice they don't want that.
- Without `set -u`, a typo like `$bar_id` instead of `$bar` expands
  to empty and may silently corrupt downstream commands. Other
  languages (Python, JS strict mode, Go, Rust) reject undefined
  references at compile or run time; bash's default is the outlier.
- Without `pipefail`, `failing_cmd | tee /var/log/x` exits successfully
  even when `failing_cmd` errored.
- `IFS=$'\n\t'` defends against the dominant bash bug class: word
  splitting on spaces in filenames. It is more aggressive than just
  quoting religiously, because it catches the cases you forgot.

For one-shot deploy, build, and migration scripts where partial
completion is worse than full failure, strict mode is the
conservative default.

## The anti-strict position

`set -e` is implicit and silent. When it works, you don't notice.
When it doesn't, the script keeps going and you also don't notice.

`set -u` trips on legitimate bash idioms.

`IFS=$'\n\t'` doesn't help if you don't also quote (the canonical
counter from
[BashFAQ/050](https://mywiki.wooledge.org/BashFAQ/050)'s comments and
the Aaron Maxwell article comments), and breaks scripts that
legitimately want default-IFS splitting.

The Wooledge alternative is explicit handling:

```bash
cmd || { msg "cmd failed"; exit 1; }
[[ -n ${var:-} ]] || die "var unset"
```

Verbose, but the failures are visible and located.

Sources: [BashFAQ/105](https://mywiki.wooledge.org/BashFAQ/105),
[BashFAQ/112](https://mywiki.wooledge.org/BashFAQ/112),
[BashPitfalls/60](https://mywiki.wooledge.org/BashPitfalls#pf60).

## The blind spots, with examples

### `set -e` is suspended in conditional contexts

```bash
set -e

if some_function; then       # all of some_function runs without -e
  echo ok
fi

some_function && true        # same: -e off inside the function

some_function || handle_err  # -e off inside the function
```

The shell behaviour is documented but counter-intuitive: once a
command is "tested" (in an `if`, `&&`, `||`, `!`, `while`, `until`
condition), `set -e` is suspended for that command *and any commands
inside it*. So `some_function` runs to completion regardless of
internal failures, and only its overall exit status matters.

ShellCheck's opt-in `check-set-e-suppressed`
(`-o check-set-e-suppressed`) catches this case.

### `set -e` is suspended in command substitution

```bash
set -e
foo=$(failing_cmd)       # script does NOT exit on bash < 4.4 default
                         # bash 4.4+ with shopt -s inherit_errexit DOES
```

Workaround: `shopt -s inherit_errexit` (bash 4.4+) makes the
substitution honour `-e`.

### `local var=$(cmd)` masks the exit status

```bash
set -e
f() {
  local x=$(failing_cmd)   # local returns 0; -e doesn't fire
  echo "$x"                # runs even though failing_cmd failed
}
```

Fix: separate declaration and assignment.

```bash
local x
x=$(failing_cmd)           # now -e fires correctly
```

Source: [SC2155](https://www.shellcheck.net/wiki/SC2155).

### `pipefail` interacts with `head`-like consumers

```bash
set -e -o pipefail
producer | head -n1        # SIGPIPE in producer; pipefail trips
```

`head` closes the pipe after one line. `producer` gets SIGPIPE,
exits non-zero, `pipefail` reports failure, `set -e` exits. The
script aborts on a correct pipeline.

Mitigations: tolerate it (`producer | head -n1 || true`), suppress
SIGPIPE handling in the producer, or restructure.

### `set -u` trips on empty arrays before bash 4.4

```bash
set -u
arr=()
echo "${arr[@]}"           # bash < 4.4: aborts with unbound variable
                           # bash 4.4+: prints nothing
```

Defensive form: `echo "${arr[@]+"${arr[@]}"}"`. Ugly.

### `set -u` and legitimate optional parameters

```bash
set -u
greet() {
  local name=$1            # aborts if called with no args
}
```

Use parameter-expansion defaults instead of bare `$1`:

```bash
greet() {
  local name=${1:-world}
}
```

Or `${1:?message}` to abort with an explicit message rather than
the cryptic "unbound variable".

### Arithmetic returning zero

```bash
set -e
let i++                    # if i was 0, let returns 1; script exits
(( i = 0 ))                # same: (( )) of zero is non-zero exit
```

Workaround: `(( i = 0 )) || true`, or use `i=$((i + 1))`.

## Opt-in shellcheck checks for strict mode

Run with `shellcheck -o name` (or `-o all` to enable everything):

| Opt-in | What it catches |
|--------|-----------------|
| `check-set-e-suppressed` | functions called in conditional contexts that silently disable `set -e` for their body |
| `check-extra-masked-returns` | command substitutions that mask `set -e` (`rm -r "$(cmd)/path"`) |
| `quote-safe-variables` | unquoted vars even when they happen to be safe (catches typos that became unsafe later) |
| `require-double-brackets` | flags `[ ]` in bash/ksh scripts |
| `require-variable-braces` | flags `$var` without `${var}` |
| `check-unassigned-uppercase` | uppercase variables that look exported but aren't |

`check-set-e-suppressed` is the most useful one for strict-mode
review specifically. It surfaces blind spots in scripts that use
`set -e` and assume it covers function bodies.

Run `shellcheck --list-optional` for the complete list with examples.

## Migration patterns

### Adding strict mode to a working script

1. Add the preamble.
2. Run the script. Likely failures: unset variables in lookups
   (`${var-}`/`${var:-}` to fix), `local x=$(cmd)` masks
   (split declaration), pipelines with `head` (add `|| true` or
   restructure), arithmetic returning zero.
3. Add `shopt -s inherit_errexit` if bash 4.4+ and the script has
   `$(…)` calls whose failures matter.
4. Walk every `if cmd; then` and `cmd && other`: confirm you actually
   want `-e` suspended inside `cmd`.

### Removing strict mode from a script that uses it

Riskier than adding. If the script's correctness depends on
implicit `-e`, removing it leaves silent failures. Add explicit
`|| { msg; exit 1; }` at each critical command first; verify
behaviour; then remove the preamble.

### Hybrid: strict at the top, relaxed in a region

```bash
set -e
critical_step

set +e
flaky_step                   # exit codes inspected manually
rc=$?
case $rc in
  0)  ok=1 ;;
  1)  retry=1 ;;
  *)  die "flaky failed: $rc" ;;
esac
set -e

next_step
```

Documents intent. Use sparingly; nested `set +e/-e` toggles become
hard to track.

## Reviewer heuristic

- Script is **short and one-shot** (build, deploy, migration): strict
  mode is defensible. Push back only if it actively breaks a known
  legitimate pattern in the script.
- Script is **long-running / interactive / library**: explicit
  handling is usually clearer than strict mode. Don't add strict
  mode mid-life; the blind spots tend to bite.
- Script **already uses strict mode**: check the documented blind
  spots (above) and flag any you find as silent fragility. Run
  `shellcheck -o check-set-e-suppressed` if you can.
- Script **already uses explicit handling**: leave it. Refactoring
  to strict mode loses information.
- Script has **no error handling at all**: the right fix is "some
  error handling" not "strict mode specifically". Either approach is
  better than none.

The decision is not "strict mode yes/no", it's "do failures get
detected somewhere". Both schools agree on that.
