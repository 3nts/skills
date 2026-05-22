# Bash idioms reference

## Contents

- Parameter expansion
- `printf -v` and string assembly
- Process substitution
- Here-strings and here-documents
- Arrays
- Functions and `local`
- Loop forms
- Brace expansion
- Case statements
- Argument parsing

## Parameter expansion

Bash's parameter expansion handles most trivial string transforms with
no fork. Reach for it before `sed`/`cut`/`tr`.

| Need | Expansion |
|------|-----------|
| String length | `${#var}` |
| Default if unset/empty | `${var:-default}` |
| Default and assign | `${var:=default}` |
| Error if unset | `${var:?message}` |
| Use alternate if set | `${var:+alt}` |
| Strip shortest prefix | `${var#pattern}` |
| Strip longest prefix | `${var##pattern}` |
| Strip shortest suffix | `${var%pattern}` |
| Strip longest suffix | `${var%%pattern}` |
| Replace first | `${var/old/new}` |
| Replace all | `${var//old/new}` |
| Replace at start | `${var/#old/new}` |
| Replace at end | `${var/%old/new}` |
| Substring | `${var:offset:length}` |
| Uppercase first | `${var^}` |
| Uppercase all | `${var^^}` |
| Lowercase first | `${var,}` |
| Lowercase all | `${var,,}` |
| Indirect | `${!varname}` |
| Array indices | `${!arr[@]}` |
| Array length | `${#arr[@]}` |

Examples:

```bash
filename="report.2026-05-21.txt"
ext="${filename##*.}"           # txt
base="${filename%.*}"           # report.2026-05-21
date_part="${filename%.*}"
date_part="${date_part##*.}"    # 2026-05-21

path="/usr/local/bin/cmd"
dir="${path%/*}"                # /usr/local/bin (no fork; beats dirname)
file="${path##*/}"              # cmd          (no fork; beats basename)
```

Source: [BashFAQ/073](https://mywiki.wooledge.org/BashFAQ/073),
[BashFAQ/100](https://mywiki.wooledge.org/BashFAQ/100),
[Google s8.2](https://google.github.io/styleguide/shellguide.html#s8.2-builtin-commands-vs-external-commands).

## `printf -v` and string assembly

`printf -v var FORMAT ARGS` writes the formatted output into `var`
without spawning a subshell. Faster than `var=$(printf …)`.

```bash
printf -v ts '%(%Y-%m-%dT%H:%M:%S)T' -1
printf -v line '%s,%s,%s' "$a" "$b" "$c"
printf -v padded '%05d' "$n"

# Join an array with commas:
printf -v csv '%s,' "${arr[@]}"
csv="${csv%,}"
```

## Process substitution

`< <(cmd)` makes a command's stdout look like a file. `> >(cmd)` makes
a command's stdin look like a file. Solves several problems at once:

- The subshell-in-piped-loop problem ([correctness.md](correctness.md#subshells-and-pipelines)):
  ```bash
  while read -r line; do count=$((count + 1)); done < <(grep foo file)
  echo "$count"   # works; would lose count with a pipe
  ```
- `diff` of two command outputs without temp files:
  ```bash
  diff <(sort file1) <(sort file2)
  ```
- `tee` to multiple processes:
  ```bash
  cmd | tee >(compressor1 > a.gz) >(compressor2 > b.bz2) >/dev/null
  ```

## Here-strings and here-documents

- **Here-string `<<<`** for short literal input. Replaces
  `echo "$x" | cmd`:
  ```bash
  read -r a b c <<< "$line"
  jq '.field' <<< "$json"
  ```
- **Here-document `<<EOF`** for multi-line literal blocks. Quote the
  delimiter (`<<'EOF'`) to disable variable expansion inside.
- **Tab-stripping `<<-EOF`** allows tab-indented bodies, with the
  indentation removed from each line. Only tabs are stripped, not
  spaces. Source: Google s5.1.

## Arrays

- **Indexed array:** `arr=(a b c)`, `arr[3]=d`, `"${arr[@]}"`.
- **Associative array:** `declare -A hash; hash[key]=value`. Bash 4+.
- **Append:** `arr+=("item")`. (Not `arr=("${arr[@]}" "item")`, which
  re-builds.)
- **Length:** `${#arr[@]}`.
- **Non-empty test:** `(( ${#arr[@]} ))`.
- **Iterate:** `for x in "${arr[@]}"; do …; done`.
- **Iterate with index:** `for i in "${!arr[@]}"; do …; done`.
- **Slice:** `"${arr[@]:1:3}"` (elements 1 through 3).
- **Delete element:** `unset 'arr[2]'` (quote the index to avoid glob
  expansion: [SC2184](https://www.shellcheck.net/wiki/SC2184)).
- **Read file into array:** `mapfile -t arr < "$file"`.
- **Read command output into array:** `mapfile -t arr < <(cmd)`.
- **NUL-delimited input (for filenames):**
  `mapfile -d '' arr < <(find … -print0)` (bash 4.4+).

Sources: [BashFAQ/005](https://mywiki.wooledge.org/BashFAQ/005),
[Google s6.7](https://google.github.io/styleguide/shellguide.html#s6.7-arrays).

## Functions and `local`

- **`name() { … }`** syntax. Skip the `function` keyword; using both
  (`function name() { … }`) is non-portable. Be consistent. Source:
  Google [s7.1](https://google.github.io/styleguide/shellguide.html#s7.1-function-names),
  [BashPitfalls/25](https://mywiki.wooledge.org/BashPitfalls#pf25).
- **`local`** every variable that shouldn't leak to the caller.
  Required for re-entrancy and prevents accidental global mutation.
  Google [s7.5](https://google.github.io/styleguide/shellguide.html#s7.5-use-local-variables).
- **`local x; x=$(cmd)`** separately, not `local x=$(cmd)`, to avoid
  masking `cmd`'s exit code. [SC2155](https://www.shellcheck.net/wiki/SC2155).
- **Return data via stdout or a named output variable,** not by
  setting globals. The named-output pattern:
  ```bash
  upper() {
    local _out=$1 _in=$2
    printf -v "$_out" '%s' "${_in^^}"
  }
  upper result "hello"   # $result is now "HELLO"
  ```
- **`return` for exit codes (0-255 only),** `exit` for terminating the
  script. [SC2151](https://www.shellcheck.net/wiki/SC2151).
- **`main "$@"` as last line** for scripts with multiple functions.
  Source: Google [s7.7](https://google.github.io/styleguide/shellguide.html#s7.7-main).

## Loop forms

- **C-style for:** `for (( i = 0; i < n; i++ )); do …; done`. Use over
  `seq` (external) or brace expansion when the bound is variable.
  Source: [BashPitfalls/33](https://mywiki.wooledge.org/BashPitfalls#pf33).
- **Glob iteration:** `for f in *.txt; do …; done`. Add
  `shopt -s nullglob` if you want zero matches to skip the loop
  instead of running once with the literal `*.txt`.
- **Whitespace-safe `find`:**
  ```bash
  while IFS= read -r -d '' f; do
    …
  done < <(find … -print0)
  ```
  Or `find … -exec foo {} +` (single foo invocation per batch).
- **`mapfile` then iterate** when the input is bounded and array
  semantics help:
  ```bash
  mapfile -t lines < "$file"
  for line in "${lines[@]}"; do …; done
  ```
- **`select`** for interactive numbered menus:
  ```bash
  select choice in apple banana exit; do
    [[ $choice == exit ]] && break
    echo "$choice"
  done
  ```

## Brace expansion

Pre-expansion, evaluated before variables. Source:
[BashPitfalls/33](https://mywiki.wooledge.org/BashPitfalls#pf33).

- **Lists:** `{a,b,c}` produces `a b c`.
- **Ranges:** `{1..10}`, `{a..z}`, `{1..10..2}` (step).
- **Combinations:** `{a,b}{1,2}` produces `a1 a2 b1 b2`.
- **Variables don't work in ranges:** `{1..$n}` is literal because
  brace expansion happens before variable expansion. Use C-style
  `for` instead.
- **Useful for backups:** `cp file.conf{,.bak}` becomes
  `cp file.conf file.conf.bak`.

## Case statements

`case` is faster and clearer than chained `if [[ ]]` for matching
against patterns. Supports the same globs as `[[ x == pattern ]]`.

```bash
case "$action" in
  start|up)
    start_service
    ;;
  stop|down)
    stop_service
    ;;
  restart|reload)
    stop_service
    start_service
    ;;
  *.gz)
    handle_compressed "$action"
    ;;
  *)
    echo "Usage: $0 {start|stop|restart}" >&2
    exit 1
    ;;
esac
```

Quote the `case` operand (`"$action"`) to prevent word splitting.
Don't quote the patterns; they need to glob. Source:
[Google s5.5](https://google.github.io/styleguide/shellguide.html#s5.5-case-statement).

## Argument parsing

- **`getopts`** for short options (POSIX-portable, no GNU long-opt
  support). Source:
  [BashFAQ/035](https://mywiki.wooledge.org/BashFAQ/035).
  ```bash
  while getopts ":vf:o:" opt; do
    case "$opt" in
      v) verbose=1 ;;
      f) input=$OPTARG ;;
      o) output=$OPTARG ;;
      :) echo "Option -$OPTARG needs an argument" >&2; exit 1 ;;
      \?) echo "Unknown option -$OPTARG" >&2; exit 1 ;;
    esac
  done
  shift $((OPTIND - 1))
  ```
- **`getopt(1)` from util-linux** for GNU long options. The BSD
  `getopt` doesn't support them; the util-linux one does.
- **Manual `while [[ $# -gt 0 ]]`** for simple long-option support
  without external tools:
  ```bash
  while [[ $# -gt 0 ]]; do
    case "$1" in
      --verbose) verbose=1; shift ;;
      --output) output=$2; shift 2 ;;
      --output=*) output=${1#*=}; shift ;;
      --) shift; break ;;
      -*) echo "Unknown option: $1" >&2; exit 1 ;;
      *) args+=("$1"); shift ;;
    esac
  done
  ```

## Useful builtins worth knowing

- **`coproc`** for two-way pipes to a child process.
- **`compgen`** for completion-style listing of builtins, functions,
  commands.
- **`command -v cmd`** to test for command existence (over `which`,
  which is external and non-standard). Source:
  [SC2230](https://www.shellcheck.net/wiki/SC2230).
- **`type cmd`** for "what kind of thing is `cmd`" (alias, function,
  builtin, file).
- **`hash`** maintains the command lookup cache.
- **`wait`** for backgrounded jobs; `wait -n` returns when any one
  finishes (bash 4.3+).

## Related

- [correctness.md](correctness.md): the bug-causing patterns these
  idioms replace.
- [performance.md](performance.md): why builtins beat external calls.
