# Wooledge BashPitfalls quick index

The high-impact entries from
[Wooledge's BashPitfalls](https://mywiki.wooledge.org/BashPitfalls)
(65 total), grouped by theme. The 40-odd entries not listed here
cover exotic edge cases (UTF-8 BOM in script files, `expr` keyword
collisions, `ed` BRE limits, `xargs -P` output races, bash 5.0-5.3-rc1
NUL-delimited-read bugs, etc.) that almost never come up in normal
review; for those, follow the URL to the full Wooledge catalogue.

Format: `[N]` is BashPitfalls/N. SC codes shown where they overlap.

## Contents

- Quoting and word splitting
- Test commands and conditionals
- Subshells and pipelines
- printf and format strings
- Iteration and reading
- Redirection and sudo
- Tilde and expansion ordering
- Arrays and commands-in-variables
- Strict mode
- The other 40+ entries

## Quoting and word splitting

- **[2]** `cp $file $target`: unquoted variable word-splits and globs.
  `cp "$file" "$target"`. (SC2086)
- **[4]** `[ $foo = "bar" ]`: unquoted in `[ ]` syntax-errors on empty
  or whitespace. Quote, or use `[[ ]]`. (SC2070)
- **[14]** `echo $foo`: same word-split-and-glob. `echo "$foo"` or
  `printf '%s\n' "$foo"`. (SC2086)
- **[24]** `for arg in $*`: use `for arg in "$@"`. (SC2048)
- **[34]** `[[ $foo = $bar ]]`: unquoted RHS is glob match. Quote RHS
  for literal compare: `[[ $foo = "$bar" ]]`. (SC2053)
- **[35]** `[[ $foo =~ 'regex' ]]`: quoted regex matches literally.
  Use `pat='regex'; [[ $foo =~ $pat ]]`. (SC2076)
- **[36]** `[ -n $foo ]`: unquoted `$foo` collapses to `[ -n ]` which
  is true. Quote or use `[[ ]]`. (SC2070)

## Test commands and conditionals

- **[7]** `[[ $foo > 7 ]]`: `>` is string compare. Use
  `(( foo > 7 ))` or `[[ $foo -gt 7 ]]`. (SC2071)
- **[19]** `cd /foo; bar`: cd may fail; chain `|| exit`. (SC2164)
- **[22]** `cmd1 && cmd2 || cmd3` is not if-then-else: if `cmd2`
  fails, `cmd3` runs. Use real `if`/`else`. (SC2015)
- **[25]** `function foo()`: mixing `function` and `()` is
  non-portable. Pick `name() { … }`. Note: shellcheck does NOT flag
  this under `-s bash` (bash accepts both); SC2112 fires only under
  `-s sh`. The recommendation is a Wooledge/Google convention.

## Subshells and pipelines

- **[8]** `cmd | while read line; do n=$((n+1)); done`: piped loop
  runs in subshell; `n` is lost. Use process substitution:
  `done < <(cmd)`.
  ([BashFAQ/024](https://mywiki.wooledge.org/BashFAQ/024))
- **[13]** `cat file | sed s/x/y/ > file`: read-and-write in pipeline
  truncates before sed reads. Use `sed -i` or temp file. (SC2094)
- **[27]** `local var=$(cmd)`: `local` masks `cmd`'s exit status.
  Split: `local var; var=$(cmd)`. (SC2155)
- **[41]** `content=$(<file)`: command substitution strips trailing
  newlines. If preservation matters, use `IFS= read -rd '' content`
  plus a marker.
- **[63]** `while … done <<< "$(cmd)"`: here-string buffers entire
  output. Stream with `< <(cmd)` instead.

## printf and format strings

- **[32]** `printf "$foo"`: format-string injection. Use
  `printf '%s\n' "$foo"`. (SC2059)

## Iteration and reading

- **[1]** `for f in $(ls *.mp3)`: word-splits + globs. Use direct
  glob: `for f in *.mp3` with `shopt -s nullglob` for empty-match
  safety. (SC2045 for `for f in $(ls)`)
- **[33]** `for i in {1..$n}`: brace expansion runs before variable
  expansion; this is literal. Use `for ((i=1; i<=n; i++))`.

## Redirection and sudo

- **[43]** `cmd 2>&1 >>logfile`: order matters. To get both to file,
  `cmd >>logfile 2>&1`. (SC2069)
- **[52]** `find . -exec sh -c 'echo {}' \;`: filenames interpolate
  into the shell command. Use the args form:
  `find . -exec sh -c 'echo "$1"' _ {} \;`. (SC2156)
- **[53]** `sudo cmd > /file`: redirection runs as current user, not
  root. `sudo sh -c 'cmd > /file'` or `cmd | sudo tee /file`. (SC2024)
- **[56]** `xargs` without `-0`: mangles whitespace in filenames. Use
  `find … -print0 | xargs -0` or `find … -exec … +`. (SC2038)

## Tilde and expansion ordering

- **[26]** `echo "~"`: tilde does not expand inside quotes. Use
  `"$HOME"` or leave `~` unquoted. (SC2088)

## Arrays and commands-in-variables

- **[50]** `args="$x -s $subj"; mail $args`: storing commands in
  strings is the single most common bash anti-pattern. Use arrays:
  `args=(-s "$subj"); mail "${args[@]}"`. Canonical source:
  [BashFAQ/050](https://mywiki.wooledge.org/BashFAQ/050).

## Strict mode

- **[60]** `set -euo pipefail`: silent blind spots in `set -e`,
  `pipefail`-vs-head interactions, `set -u` empty-array tripping.
  Full debate, blind spots, opt-in shellcheck checks:
  [strict-mode.md](strict-mode.md).

## The other 40+ entries

Exotic patterns and edge cases worth knowing exist but rarely come
up. To name a few: `tr [A-Z][a-z]` locale issues ([30]), broken
symlinks with `-e` ([37]), `expr` keyword collisions ([39]),
UTF-8 BOM in script files ([40], also SC1082),
`xargs -P` output interleaving ([51]), closing stderr vs redirecting
([55] = `2>&-` vs `2>/dev/null`),
`unset a[0]` glob expansion ([57], also SC2184),
two `date` calls straddling midnight ([58]),
arithmetic injection via array subscripts ([45], [46], [62]),
NUL-delimited `read` bugs in bash 5.0-5.3-rc1 UTF-8 locales ([65]).

If a script uses something you don't recognize and don't see here,
check [the full Wooledge BashPitfalls page](https://mywiki.wooledge.org/BashPitfalls).
The page is stable; the numbering doesn't change.

## How to use this file

- **Cite by number** in a review: "this is BashPitfalls/8" is
  unambiguous and verifiable in seconds.
- **Scan checklist** when reviewing a script: walk top-to-bottom
  asking "does any of this apply here?"
- **For deeper write-ups** of any entry, the canonical URL is
  `https://mywiki.wooledge.org/BashPitfalls#pfN`.
