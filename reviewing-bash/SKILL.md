---
name: reviewing-bash
description: >
  Reviews bash and shell scripts (.sh, sh, bash) for correctness, idiomatic
  style, performance, and ShellCheck compliance. Use when the user asks to
  review, refactor, audit, lint, polish, improve, clean up, fix, or
  make-better a shell script; or asks "what's wrong with this", "is this
  idiomatic", "take a look at this bash", "this script is ugly", "any
  feedback on this", "tighten this up". Cites Wooledge BashFAQ /
  BashPitfalls / BashGuide, Google Shell Style Guide, ShellCheck wiki (SC
  codes), and shfmt. Produces grouped findings (real bugs, idioms,
  stylistic) with rule citations and an optional refactored version. Scope
  is review and refactor of existing scripts, not greenfield authoring;
  bash primarily, sh portability noted where it intersects.
allowed-tools: Bash(shellcheck *) Bash(shfmt *) Bash(bash -n *) Bash(bash -x *) Read Grep Edit
---

# Reviewing bash

For reviewing and refactoring existing bash or POSIX-sh scripts. This file
holds the review process, the high-frequency rules, and the open debates.
Catalogues and deeper rationale are one level deep under `reference/`.

## Process

1. **Read the whole script first.** Don't comment on line N before you've
   seen line M+1. Note the shebang, the entry point, the inputs, the
   outputs, and the failure modes.
2. **Run the tools** if installed (see "Tooling" below):
   - `bash -n <path>` (parse-only syntax check)
   - `shellcheck -s bash <path>` (static analyser)
   - `shfmt -d <path>` (formatter diff)
   Treat tool output as signal, not as the review. Tools miss design
   problems; reviewers miss what tools catch.
3. **Walk findings in tiers.** Group by impact, in this order:
   - **Real bugs.** Will misbehave on plausible inputs.
   - **Fragility.** Will misbehave on edge inputs (spaces in filenames,
     empty arrays, broken symlinks, UTF-8 surprises).
   - **Idioms.** Verbose where bash has a one-liner.
   - **Style.** Won't change behaviour; lowers the cost of the next read.
4. **Cite the rule** for each non-obvious finding. SC code, BashFAQ
   number, BashPitfalls number, or Google section anchor. Cited findings
   are auditable; uncited ones are opinion.
5. **Propose fixes as a diff or replacement snippet,** not as prose.
   Show the corrected line beside the original.
6. **Push back when the script is fine.** A clean shellcheck run plus
   "does what it should" is a finished review. Don't manufacture work.

## Output format

Two parts, in this order:

1. **Findings list,** grouped by impact tier. One entry per finding:
   `file:line | issue | fix in shorthand | citation`.
2. **The refactored script,** only if the user asked for a refactor (not
   just a review). Show as a unified diff or a full file, whichever is
   shorter.

Don't narrate the review chronologically. Don't restate the script back
to the user. Don't recommend `set -euo pipefail` reflexively (see "Error
handling" below).

## High-impact rules

The patterns that come up in almost every bash review. Detailed coverage
in `reference/`; the headlines are here.

### Quoting

- **Quote every parameter expansion.** `"$var"`, `"${arr[@]}"`. Unquoted
  expansion undergoes word splitting on `IFS` and pathname expansion.
  Source: [SC2086](https://www.shellcheck.net/wiki/SC2086),
  [BashPitfalls/1](https://mywiki.wooledge.org/BashPitfalls#pf1),
  [Google s5.7](https://google.github.io/styleguide/shellguide.html#s5.7-quoting).
- **`"$@"` not `$*` or `$@`.** `$*` joins on first IFS char; unquoted
  `$@` word-splits each element. Source:
  [SC2068](https://www.shellcheck.net/wiki/SC2068),
  [BashPitfalls/24](https://mywiki.wooledge.org/BashPitfalls#pf24).
- **Inside `[[ ]]`, quote the RHS of `=` for literal compare;** leave it
  unquoted for glob match. The choice is meaningful, not stylistic.
  Source: [BashPitfalls/34](https://mywiki.wooledge.org/BashPitfalls#pf34),
  [SC2053](https://www.shellcheck.net/wiki/SC2053).
- More in [reference/correctness.md](reference/correctness.md#quoting).

### Test commands

- **`[[ ]]` over `[ ]` in bash.** `[[` is a keyword (no word splitting,
  no glob expansion of operands, supports `=~` and pattern matching).
  Source:
  [Google s6.3](https://google.github.io/styleguide/shellguide.html#s6.3-tests),
  [BashFAQ/031](https://mywiki.wooledge.org/BashFAQ/031).
- **`(( ))` for integer compare;** `<`/`>` inside `[[ ]]` are string
  ops, not numeric. Source:
  [Google s6.4](https://google.github.io/styleguide/shellguide.html#s6.4-testing-strings),
  [SC2071](https://www.shellcheck.net/wiki/SC2071).
- **Never `expr`;** use `$(( ))`. Source:
  [Google s6.9](https://google.github.io/styleguide/shellguide.html#s6.9-arithmetic),
  [SC2003](https://www.shellcheck.net/wiki/SC2003).

### Subshells and pipelines

- **Variables set in a piped loop don't survive the loop.** Each pipe
  stage runs in a subshell. Use process substitution:
  `while read -r x; do …; done < <(cmd)`. Source:
  [BashFAQ/024](https://mywiki.wooledge.org/BashFAQ/024),
  [BashPitfalls/8](https://mywiki.wooledge.org/BashPitfalls#pf8),
  [SC2030](https://www.shellcheck.net/wiki/SC2030)/[SC2031](https://www.shellcheck.net/wiki/SC2031),
  [Google s6.8](https://google.github.io/styleguide/shellguide.html#s6.8-pipes-to-while).
- **`shopt -s lastpipe`** (bash 4.2+) is the other fix; requires job
  control off.

### Reading input

- **Always `read -r`.** Without it, backslashes get mangled. Source:
  [SC2162](https://www.shellcheck.net/wiki/SC2162),
  [BashFAQ/001](https://mywiki.wooledge.org/BashFAQ/001).
- **`while IFS= read -r line; do …; done < "$file"`** is the canonical
  whole-line read. `IFS=` keeps leading/trailing whitespace.
- **`mapfile -t arr < "$file"`** or `readarray -t arr < "$file"` is
  faster and clearer than append-in-a-loop. Source: BashFAQ/001.
- **Never iterate `ls` or `find` output unquoted.** Use globs or
  `find -print0 | xargs -0` / `find -exec +`. Source:
  [SC2010](https://www.shellcheck.net/wiki/SC2010),
  [SC2044](https://www.shellcheck.net/wiki/SC2044),
  [SC2045](https://www.shellcheck.net/wiki/SC2045),
  [BashPitfalls/1](https://mywiki.wooledge.org/BashPitfalls#pf1).

### Substitutions

- **`$()` not backticks.** Backticks have weird backslash rules and
  nest badly. Source:
  [Google s6.2](https://google.github.io/styleguide/shellguide.html#s6.2-command-substitution),
  [BashFAQ/082](https://mywiki.wooledge.org/BashFAQ/082),
  [SC2006](https://www.shellcheck.net/wiki/SC2006).
- **`printf` over `echo`** for any output that might contain `\`, a
  leading `-`, or anything you want portable. `echo "$x"` is
  unspecified for many inputs; `printf '%s\n' "$x"` is always correct.
  Source: [SC2028](https://www.shellcheck.net/wiki/SC2028),
  [SC2059](https://www.shellcheck.net/wiki/SC2059),
  [BashPitfalls/32](https://mywiki.wooledge.org/BashPitfalls#pf32).
- **Never put a variable in a `printf` format string.** Pass it as an
  argument: `printf '%s\n' "$x"`, not `printf "$x\n"`. SC2059.

### Error handling: the strict-mode tension

No community consensus. Google + Aaron Maxwell say
`set -euo pipefail; IFS=$'\n\t'`; Wooledge documents the blind spots
(BashFAQ/105, /112, BashPitfalls/60). Report both positions; don't
insert strict mode reflexively; don't strip it from a script that has
it. If the script already uses `set -e`, additionally check:
`cd "$dir" || exit` ([SC2164](https://www.shellcheck.net/wiki/SC2164)),
split `local x; x=$(cmd)` ([SC2155](https://www.shellcheck.net/wiki/SC2155)),
no separate `$?` checks ([SC2181](https://www.shellcheck.net/wiki/SC2181)),
and consider `shellcheck -o check-set-e-suppressed` to surface
function-in-conditional blind spots. Full debate, the seven documented
blind spots, and reviewer heuristic in
[reference/strict-mode.md](reference/strict-mode.md).

### `cd` safety

- **`cd "$dir" || exit`** (or `|| return` inside a function). Silent
  `cd` failure runs subsequent commands in the wrong directory; with
  `rm -rf` somewhere downstream, this is destructive. Source:
  [SC2164](https://www.shellcheck.net/wiki/SC2164).

### eval and code-as-data

- **Avoid `eval`.** If you're tempted, you probably want an array
  (`args=(-s "$x")`, then `cmd "${args[@]}"`), a function, or a
  parameter expansion default. Source:
  [BashFAQ/048](https://mywiki.wooledge.org/BashFAQ/048),
  [BashFAQ/050](https://mywiki.wooledge.org/BashFAQ/050),
  [Google s6.6](https://google.github.io/styleguide/shellguide.html#s6.6-eval).
- **Variables hold data; functions hold code.** Don't store commands
  in strings. BashFAQ/050 is the canonical reference.

## Tier 2: idioms, style, performance

These tiers mark fluency, not correctness. Skip them unless the user
asked for a full polish.

- **Idioms:** parameter expansion over external tools
  (`${var%.txt}`, `${var//old/new}`, `${var:-default}`), `printf -v`
  for no-fork string assembly, `mapfile -t arr < file`, process
  substitution `< <(cmd)` to dodge subshell-in-piped-loop, here-strings
  `cmd <<< "$x"`. Full set in [reference/idioms.md](reference/idioms.md).
- **Style:** `#!/usr/bin/env bash`; 2-space indent
  ([Google s5.1](https://google.github.io/styleguide/shellguide.html#s5.1-indentation));
  80-char target; `name() { … }` syntax (consistent throughout);
  lowercase locals/functions, ALL_CAPS exported constants; comments
  document why, not what. Full guidance in
  [reference/style.md](reference/style.md).
- **Performance:** every external command forks (~1ms Linux, ~3-5ms
  macOS); builtins are microseconds. Hoist external calls out of
  loops; prefer one `jq`/`awk`/`sed` over N. `cat file | cmd` is
  wasted overhead ([SC2002](https://www.shellcheck.net/wiki/SC2002)
  is opt-in via `-o useless-use-of-cat`; flag in review regardless).
  Full guidance in [reference/performance.md](reference/performance.md).

## When not to use bash

A correct review sometimes says "this should be rewritten in another
language." Signals: floating-point math, JSON/XML/HTML parsing,
record-oriented data, anything > 100 lines with non-linear control
flow, anything that needs exceptions or proper data structures.
Source: [Wooledge BashWeaknesses](https://mywiki.wooledge.org/BashWeaknesses),
[Google s1.2](https://google.github.io/styleguide/shellguide.html#s1.2-when-to-use-shell).
Full list in [reference/when-not-to-use-bash.md](reference/when-not-to-use-bash.md).

Cross-platform deployment (macOS, BSD, Alpine, BusyBox) has its own
constraints; see [reference/portability.md](reference/portability.md).

## Tooling

### shellcheck

Static analyser. `shellcheck -s bash <path>` is the baseline (the
dialect must match the shebang: SC3xxx codes are real bugs under
`#!/bin/sh` and noise under `#!/usr/bin/env bash`). Severity tiers:
SC1xxx (parser, blocking), SC2xxx (runtime/logic, most reviews),
SC3xxx (POSIX portability).

**Opt-in checks** worth enabling explicitly with `-o name` (or `-o all`):
`useless-use-of-cat` (SC2002), `check-set-e-suppressed`,
`check-extra-masked-returns`, `quote-safe-variables`,
`require-double-brackets`, `require-variable-braces`,
`deprecate-which`, `check-unassigned-uppercase`, `add-default-case`,
`avoid-nullary-conditions`, `avoid-negated-conditions`. List with
`shellcheck --list-optional`.

Disable individual warnings inline with `# shellcheck disable=SCnnnn`
only with a one-line justification on the next line.

If not installed:
- Debian/Ubuntu: `apt install shellcheck`
- Arch: `pacman -S shellcheck`
- macOS: `brew install shellcheck`
- Nix: `nix profile install nixpkgs#shellcheck`

Top codes by frequency: see
[reference/shellcheck-codes.md](reference/shellcheck-codes.md).

### shfmt

Formatter. `shfmt -d <path>` shows the diff; `shfmt -w <path>` writes
it. Reasonable defaults: `shfmt -i 2 -ci -sr -bn <path>`
(2-space indent, case-indent, space-after-redirect, binary-on-next-
line). The project deliberately keeps flags minimal.

If not installed:
- Debian/Ubuntu: `apt install shfmt`
- Arch: `pacman -S shfmt`
- macOS: `brew install shfmt`
- Go: `go install mvdan.cc/sh/v3/cmd/shfmt@latest`

### bash -n / -x

`bash -n <path>` parse-only; `bash -x <path>` traces every expanded
command. Use `-x` when the script's behaviour disagrees with the
reviewer's reading of it.

If neither shellcheck nor shfmt is available, say so in the review,
fall back to `bash -n`, and apply the rules in this skill manually.

## Reference index

- [reference/correctness.md](reference/correctness.md): bug-causing
  patterns. Quoting, tests, subshells, error handling, eval, races.
- [reference/idioms.md](reference/idioms.md): fluent bash. Parameter
  expansion, `printf -v`, process substitution, mapfile, here-strings.
- [reference/style.md](reference/style.md): naming, formatting, file
  structure, comments, function placement.
- [reference/performance.md](reference/performance.md): fork cost,
  builtins-vs-externals, single-pass tools, hot-loop hoisting.
- [reference/pitfalls.md](reference/pitfalls.md): the Wooledge
  BashPitfalls catalogue (1-65), grouped by category.
- [reference/shellcheck-codes.md](reference/shellcheck-codes.md): SC
  codes most-encountered in review, with one-line cause and fix.
- [reference/strict-mode.md](reference/strict-mode.md): the
  `set -euo pipefail` debate, blind spots, opt-in shellcheck checks,
  migration patterns.
- [reference/portability.md](reference/portability.md): bash version
  differences, POSIX sh restrictions, GNU vs BSD coreutils, macOS
  gotchas, BusyBox.
- [reference/example-review.md](reference/example-review.md): one
  full worked review with real shellcheck and shfmt output,
  demonstrating the SKILL.md output format.
- [reference/when-not-to-use-bash.md](reference/when-not-to-use-bash.md):
  BashWeaknesses plus signals for switching language.

## Primary sources

- Wooledge: [BashFAQ](https://mywiki.wooledge.org/BashFAQ),
  [BashPitfalls](https://mywiki.wooledge.org/BashPitfalls),
  [BashGuide/Practices](https://mywiki.wooledge.org/BashGuide/Practices),
  [BashWeaknesses](https://mywiki.wooledge.org/BashWeaknesses),
  [Quotes](https://mywiki.wooledge.org/Quotes).
- Google:
  [Shell Style Guide](https://google.github.io/styleguide/shellguide.html).
- ShellCheck: [wiki](https://www.shellcheck.net/wiki/),
  [Checks](https://github.com/koalaman/shellcheck/wiki/Checks).
- shfmt: [github.com/mvdan/sh](https://github.com/mvdan/sh),
  [manpage](https://github.com/mvdan/sh/blob/master/cmd/shfmt/shfmt.1.scd).
- Strict mode debate: Aaron Maxwell,
  [Unofficial Bash Strict Mode](http://redsymbol.net/articles/unofficial-bash-strict-mode/);
  counter-position in BashFAQ/105 and BashFAQ/112.
