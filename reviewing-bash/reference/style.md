# Bash style reference

## Contents

- Shebang and file header
- Indentation and line length
- Function declaration syntax
- Naming conventions
- File structure
- Comments
- Pipeline and control-flow formatting
- `case` statement formatting
- shfmt defaults

## Shebang and file header

- **`#!/usr/bin/env bash`** for portability. Google
  ([s1.1](https://google.github.io/styleguide/shellguide.html#s1.1-which-shell-to-use))
  says `#!/bin/bash` is fine in environments where bash is guaranteed
  at that exact path. Wooledge prefers `#!/usr/bin/env bash`
  ([BashGuide/Practices](https://mywiki.wooledge.org/BashGuide/Practices)).
  Either is defensible; consistency within a codebase matters more.
- **Shebang on line 1, no blanks before it.** Source:
  [SC1128](https://www.shellcheck.net/wiki/SC1128).
- **No extension on executables.** A program invoked by name (on
  `$PATH` or directly) should be `myutil`, not `myutil.sh`. The
  suffix exposes implementation and ties callers to bash forever:
  a future rewrite in Go either renames (breaking callers) or keeps
  a lying suffix. Unix convention is `git` not `git.c`, `ls` not
  `ls.c`.
- **`.sh` is fine on libraries** (sourced files, never executed):
  editor mode detection and `find -name '*.sh'` are useful.
- Google
  [s2.1](https://google.github.io/styleguide/shellguide.html#s2.1-file-extensions)
  permits `.sh` on executables; the no-extension form is the broader
  Unix convention. If the project already uses `.sh` consistently,
  flag inconsistency rather than demanding a rename.
- **File header comment** with a one-line description of what the
  script does. Source:
  [Google s4.1](https://google.github.io/styleguide/shellguide.html#s4.1-file-header).

## Indentation and line length

- **2-space indent, no tabs.** Source:
  [Google s5.1](https://google.github.io/styleguide/shellguide.html#s5.1-indentation).
- **Exception: tabs inside `<<-` here-docs.** The `<<-` form strips
  leading tabs only.
- **80-char target line length** per Google
  [s5.2](https://google.github.io/styleguide/shellguide.html#s5.2-line-length-and-long-strings).
  Long URLs, file paths, and single-token identifiers may exceed.
  Use `\` line continuation or here-docs for long strings.

## Function declaration syntax

- **`name() { … }`**, not `function name { … }` or
  `function name() { … }`. Mixing `function` and `()` is non-portable
  ([BashPitfalls/25](https://mywiki.wooledge.org/BashPitfalls#pf25)).
  Google
  [s7.1](https://google.github.io/styleguide/shellguide.html#s7.1-function-names)
  picks `name() {`; consistency throughout the file matters.
- **No space between name and parens;** opening brace on the same
  line. (shfmt's default behaviour also moves opening braces to the
  same line.)
- Note: shellcheck does NOT flag this under `-s bash` because bash
  accepts both forms. SC2112 fires only when the dialect is `-s sh`.
  The recommendation is a Wooledge/Google convention; flag manually
  in review even when shellcheck stays silent.

## Naming conventions

| Kind | Convention | Example |
|------|------------|---------|
| Local variable | `lower_snake_case` | `local file_count=0` |
| Function | `lower_snake_case` | `process_file()` |
| Library namespace | `namespace::function` | `myutil::log_error()` |
| Constant (file scope) | `UPPER_SNAKE_CASE` | `readonly MAX_RETRIES=3` |
| Exported env var | `UPPER_SNAKE_CASE` | `export DATABASE_URL=…` |
| Loop index in tight loops | single letter is OK | `for ((i=0;i<n;i++))` |

Source:
[Google s7.1](https://google.github.io/styleguide/shellguide.html#s7.1-function-names),
[s7.2](https://google.github.io/styleguide/shellguide.html#s7.2-variable-names),
[s7.3](https://google.github.io/styleguide/shellguide.html#s7.3-constants-and-environment-variable-names).

Avoid:
- Single-letter names outside tight loops.
- Reusing builtin/env names: `PATH`, `IFS`, `HOME`, `USER`, `PWD`,
  `LINENO`, `RANDOM`, `SECONDS`, `BASH_*`. Source:
  [SC2123](https://www.shellcheck.net/wiki/SC2123).

## File structure

Recommended top-to-bottom order:

1. Shebang
2. File header comment (one-line description, copyright/author if
   relevant)
3. Constants (`readonly`, `declare -r`, `export`)
4. Library sources (`source ./lib/util.sh`)
5. Function definitions
6. `main "$@"` as the last non-comment line

Source: Google
[s7.6](https://google.github.io/styleguide/shellguide.html#s7.6-function-location),
[s7.7](https://google.github.io/styleguide/shellguide.html#s7.7-main).

A `main` function is required for scripts long enough to define
another function. Short linear scripts don't need one.

## Comments

- **File header:** one-line description of what the script does.
  Source: Google s4.1.
- **Function header** for any function that isn't trivially obvious.
  Document: purpose, globals used/modified, arguments, outputs,
  return code. Source: Google
  [s4.2](https://google.github.io/styleguide/shellguide.html#s4.2-function-comments).
- **Implementation comments** for the non-obvious. Source: Google
  [s4.3](https://google.github.io/styleguide/shellguide.html#s4.3-implementation-comments).
- **Wooledge's guidance:** "Comment your way of thinking before you
  forget."
  ([BashGuide/Practices](https://mywiki.wooledge.org/BashGuide/Practices)).
- **Default to none.** A comment that paraphrases the line above it
  is noise. A comment that explains a non-obvious decision earns its
  place.
- **TODO format:** `# TODO(name-or-email): description`. Searchable,
  attributable. Source: Google
  [s4.4](https://google.github.io/styleguide/shellguide.html#s4.4-todo-comments).

## Pipeline and control-flow formatting

- **Pipelines on one line if they fit.** Source: Google
  [s5.3](https://google.github.io/styleguide/shellguide.html#s5.3-pipelines).
- **Otherwise, split one segment per line** with the pipe at the
  start of the next line and 2-space indent. shfmt's `-bn` flag
  (binary-next-line) does this:
  ```bash
  command_one \
    | command_two \
    | command_three
  ```
- **`; then` and `; do` on the same line** as `if`/`while`/`for`:
  ```bash
  if [[ $x -gt 0 ]]; then
    …
  fi
  ```
  Source: Google
  [s5.4](https://google.github.io/styleguide/shellguide.html#s5.4-control-flow).
- **`else` on its own line.** Closing `fi`/`done` on their own line.
- **In `for`, include `in "$@"`** when iterating arguments, for
  clarity even though it's the default.

## `case` statement formatting

```bash
case "${expression}" in
  a|b)
    do_something
    ;;
  c)
    do_something_else
    ;;
  *)
    fallback
    ;;
esac
```

- Match expressions indented one level from `case`/`esac`.
- One-line alternatives: space after `)`, space before `;;`.
- Multi-line alternatives: actions indented another level, `;;` on
  its own line.
- Avoid `;&` and `;;&` (fall-through variants). Source: Google
  [s5.5](https://google.github.io/styleguide/shellguide.html#s5.5-case-statement).

## Quoting and variable expansion (style aspect)

The correctness rules are in
[correctness.md](correctness.md#quoting). Stylistic on top:

- **`"${var}"`** with explicit braces for readability when the
  variable is adjacent to other characters. `"${pre}_${post}"` is
  clearer than `"$pre"_"$post"`. Source: Google
  [s5.6](https://google.github.io/styleguide/shellguide.html#s5.6-variable-expansion).
- **Braces are not quoting.** `"${var}"` and `"$var"` quote the same
  way; the braces are for delimiting the name.
- **Positional `$1`-`$9`** don't need braces; `${10}` and up do.
- **Numeric special vars** (`$$`, `$#`, `$!`, `$?`) traditionally
  unbraced.

## shfmt defaults

`shfmt` is opinionated by design. Useful options for a review pass:

```sh
shfmt -i 2 -ci -sr -bn -d <path>
```

| Flag | Meaning |
|------|---------|
| `-i 2` | indent with 2 spaces (default is tabs) |
| `-ci` | indent switch `case` alternatives |
| `-sr` | space after redirect operators (`> file` not `>file`) |
| `-bn` | binary operators (`&&`, `\|`, `\|\|`) start the new line |
| `-s`  | simplify (apply safe rewrites like dropping `${var}` to `$var` where unambiguous) |
| `-d`  | diff mode: print diff, exit non-zero if changes needed |
| `-w`  | write the formatted output back to file |
| `-l`  | list files that differ |
| `-ln=bash\|posix\|mksh\|bats\|zsh` | dialect (default: auto) |
| `--filename name` | provide a name when reading from stdin |
| `-mn` | minify (rarely useful in review) |

Source:
[github.com/mvdan/sh](https://github.com/mvdan/sh),
[shfmt manpage](https://github.com/mvdan/sh/blob/master/cmd/shfmt/shfmt.1.scd).

A reviewer running `shfmt -d` should treat the diff as a suggestion,
not a mandate. Some projects diverge from shfmt defaults intentionally
(four-space indent, etc.); flag if inconsistent within a file, not if
the whole project is consistent.

## Related

- [correctness.md](correctness.md): rules that change behaviour.
- [idioms.md](idioms.md): fluent patterns.
