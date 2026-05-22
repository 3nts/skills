# When not to use bash

A correct review sometimes says "this should be rewritten in another
language." Bash is a glue language; outside its sweet spot it costs
more to maintain than it saves. The Wooledge
[BashWeaknesses](https://mywiki.wooledge.org/BashWeaknesses) page is
the canonical inventory; Google
[s1.2](https://google.github.io/styleguide/shellguide.html#s1.2-when-to-use-shell)
adds the operational guidance ("rewrite scripts exceeding 100 lines
or using non-straightforward control flow in a structured language").

## Contents

- Hard limits
- Soft signals
- What to switch to
- Counter-arguments and pushback
- The 100-line heuristic

## Hard limits

Bash genuinely cannot do these well. Don't try.

- **Floating-point math.** Bash has integer arithmetic only. Use `bc`,
  `awk`, `python`, or push the math into a single `jq` expression.
- **JSON parsing.** Tag-based, recursive, escaped: regex won't cut
  it. Use `jq` (and one invocation, not N). For anything beyond
  read-only `jq` queries, switch to Python/Node/Go.
- **XML / HTML parsing.** Same reason. Use `xmlstarlet`, `xsltproc`,
  Python's `lxml`, or a real HTML parser.
- **Binary data.** Bash variables can't hold NUL bytes; you can't pass
  NUL through argv. Use Python, Perl, or C.
- **Concurrent I/O across many endpoints.** Bash has no equivalent of
  `select(2)` / `poll(2)` / `epoll`. Background-job juggling with
  `&`/`wait` works for tens, breaks for hundreds, is unrecoverable at
  thousands.
- **Sorted-data manipulation.** Use `sort`, `uniq`, `comm`, `join` as
  external tools. Pure-bash sort algorithms are painful and slow.
- **Database queries beyond basic invocation.** Tuple boundaries,
  field types, transactions: use a language with the DB's API.
- **Proper exception handling and recovery.** Bash has no `try`/
  `catch`; `set -e` is implicit and silent (see
  [correctness.md](correctness.md#error-handling-the-strict-mode-tension)).
  Complex error recovery wants real exceptions.
- **Privilege dropping.** Hard to do safely from bash; the
  external `setpriv`/`runuser` tools lose environment context.
- **Records and structs.** Bash has scalars, indexed arrays, and
  associative arrays. Nesting requires gross hacks (serializing keys
  with `:` separators, named-reference indirection). At the first
  nested structure, switch languages.
- **Closures, first-class functions, lambdas.** Bash functions are
  global; arguments pass by value (no by-reference except via name
  references with their own injection caveats).

## Soft signals

When you see these in a script, ask whether bash is still the right
tool. None is fatal on its own; two or three together usually mean
yes.

- The script is > 100 lines (Google
  [s1.2](https://google.github.io/styleguide/shellguide.html#s1.2-when-to-use-shell)).
- Non-linear control flow: deep nesting, mutual recursion, state
  machines.
- Multiple processes managed concurrently with `wait`/`kill`/job
  control.
- A function returning data via global variables because
  bash-functions-can't-return-strings.
- More than two `eval` calls.
- Hand-rolled CSV/TSV/INI parsing.
- Hand-rolled HTTP retry loops with backoff (use curl's own
  `--retry` or switch language).
- Hand-rolled JSON construction with `printf` and concatenation.
- Heavy reliance on `awk -F` for record-oriented logic.
- Most lines start with `if [[ … ]]`.
- A unit-test harness for the bash itself ([bats] is fine for small
  scripts but if you're writing 100s of tests, you're past bash's
  good range).

[bats]: https://github.com/bats-core/bats-core

## What to switch to

| Problem | Better tool |
|---------|-------------|
| Text processing > one substitution per line | `awk` (still in pipeline) or Python |
| JSON read/write | `jq` for read, Python for write |
| XML/HTML | `xmlstarlet`, Python `lxml`, `pup`, `htmlq` |
| HTTP with retries, headers, JSON bodies | `curl` flags first, then Python `requests`/`httpx` |
| Concurrent I/O | Python `asyncio`, Go, Node |
| Floats / stats | `bc`, `awk`, then Python (`numpy`/`statistics`) |
| Records / nested data | Python, Go, anything typed |
| Long-running daemon | Python, Go, Rust |
| CLI with subcommands and parsed args | Python (`argparse`/`click`/`typer`), Go (`cobra`) |
| Cross-platform (Windows included) | Python; portable POSIX `sh` is hostile to Windows |

The migration is usually painless: pull the script's external commands
out (jq, curl, find) and keep them; replace the bash glue. The
external tools were doing the real work; bash was the wiring.

## Counter-arguments and pushback

Push back on a "rewrite in Python" recommendation when:

- The script is the deployment target (CI runner, build hook,
  container entrypoint, systemd ExecStart). Python adds a runtime
  dependency that bash doesn't. Plain bash + coreutils ships
  everywhere.
- It's < 50 lines, linear, and the author handles errors explicitly.
  Adding Python's overhead would be net negative.
- It's primarily a sequence of CLI invocations with light flow
  control. That's exactly what bash is for.
- The team writes bash fluently and rewrites to a language they
  write less fluently would lose more than it gains.

The decision is "is bash still cheaper than the alternative" not "is
this above some line count." 200 lines of well-written bash beats
200 lines of poorly-written Python.

## The 100-line heuristic

Google's
[s1.2](https://google.github.io/styleguide/shellguide.html#s1.2-when-to-use-shell)
guidance: "Rewrite scripts exceeding 100 lines or using
non-straightforward control flow in a structured language."

Treat 100 lines as a prompt to evaluate, not as a rule. Some scripts
remain readable at 300 lines (a long sequence of clearly-named
function calls). Others demand a rewrite at 50 lines (deeply nested
conditionals on parsed input).

When recommending a rewrite, name the specific weakness that triggered
the recommendation (data structures, concurrency, parsing, etc.). A
review finding of "this is too long, rewrite in Python" is unhelpful;
"this hand-parses JSON with `cut` and `awk`, will break on nested
objects; Python with `json.load` is one line" is actionable.
