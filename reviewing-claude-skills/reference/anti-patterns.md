# Skill anti-patterns

## Contents

- 1. Length (SKILL.md > 500 lines, reference > 100 without TOC)
- 2. Weak description (missing WHEN or negative triggers)
- 3. Description overflow (combined > 1,536 chars)
- 4. Meta-narration in SKILL.md body
- 5. Human-reader content
- 6. Orphaned reference files
- 7. Description-body mismatch
- 8. Should-be-a-subagent
- 9. Over-broad `allowed-tools`
- 10. Side-effect skill auto-invocable

## 1. Length

**Signals:** SKILL.md exceeds 500 lines; any `reference/*.md` exceeds 100
lines without a `## Contents` heading near the top.

**Why it bites:** SKILL.md body stays in context across turns once invoked,
so every line is recurring cost. Reference files >100 lines without a TOC
mean Claude can't preview scope before committing the full read.

**Fix:** Split SKILL.md by topic into `reference/*.md`. Add a `## Contents`
list to any reference file over 100 lines. Anthropic's own guideline.

## 2. Weak description

**Signals:** Description names *what* the skill does but not *when* to
invoke it; no trigger phrases (file patterns like `**/*.tf`, symptoms like
"loop scope errors", task verbs like "review", "audit", "lint"); no
negative triggers ("skip for X").

**Why it bites:** Claude uses the description for auto-activation. Without
WHEN, the skill is invisible to the matcher. Without negative triggers, it
over-fires on prompts that merely sound related.

**Fix:** Restructure as: `<WHAT>. Use when <triggers>. Skip for <negative
triggers>.` Lead with the key use case — combined `description` +
`when_to_use` is truncated at 1,536 chars and trailing content goes first.

## 3. Description overflow

**Signals:** Combined `description` + `when_to_use` exceeds 1,536 chars
(`/doctor` flags listing budget overflow); negative triggers visibly
truncated in the `/skills` menu.

**Why it bites:** Truncation drops the tail, often where negative triggers
or fine-grained activation rules live, leading to over-firing.

**Fix:** Compress. Key use case first, trigger keywords second, negative
triggers third. If still too long, condense further — `description` is
*activation metadata*, not documentation. Ruthless brevity wins.

## 4. Meta-narration in SKILL.md body

**Signals:** Paragraphs that describe the file itself rather than instruct
the agent. Common forms:

- "This file holds the high-frequency content; details are under `reference/`."
- "Below you'll find the design principles, followed by anti-patterns."
- "This skill is organized into three sections..."

**Why it bites:** Recurring token cost for content that gives the agent no
new capability. Section headings already carry navigation.

**Fix:** Delete. `## Design principles`, `## Anti-pattern scan` etc. are
self-evident from headings; prose intros are redundant.

## 5. Human-reader content

**Signals:** `## Sources`, `## Credits`, `## Changelog`, `## Acknowledgments`
sections; attributions to people or books not actively cited in agent-facing
guidance; "v2.1 — added X" history lines.

**Why it bites:** Both SKILL.md and `reference/*.md` are agent prompt.
Credits don't change Claude's behavior — they're recurring token cost in
SKILL.md, or on-demand-loaded waste in reference files.

**Fix:** Delete from the skill. If provenance matters for the human author,
keep it in git history (commit messages, PR descriptions) or a sibling
README that lives next to but outside the skill directory. Don't relocate
to `reference/` as a compromise — reference/ is still agent prompt.

## 6. Orphaned reference files

**Signals:** `reference/foo.md` exists on disk but is not mentioned anywhere
in SKILL.md.

**Why it bites:** Claude discovers reference files through SKILL.md links.
An unlinked file is invisible — the agent won't know the file exists or
what it contains, so the content never gets loaded.

**Fix:** Add a one-line entry under `## Deeper references` (or equivalent)
in SKILL.md describing what's in the file and when to load it. If you
can't write that line, the file probably shouldn't exist.

## 7. Description-body mismatch

**Signals:** Description claims the skill covers X, but the body covers Y.
Or trigger phrases in the description ("fixture-scope issues") don't have
corresponding content in the body.

**Why it bites:** Skill auto-fires on prompts matching the description,
then the agent reads body content that doesn't help with the user's actual
problem — wasting tokens and producing irrelevant guidance.

**Fix:** Re-derive the description from the body. List every section
heading and confirm each is implied by something in the description. If a
section has no description hook, either add the hook or cut the section.

## 8. Should-be-a-subagent

**Signals:** Skill body reads like a system prompt for an autonomous
worker ("you are a senior reviewer..."). The "task" is self-contained
("scan the codebase and return findings"). Output is summarized and
discarded rather than applied in-place. The skill needs tool restrictions
to be safe.

**Why it bites:** Skills run in the main thread's context. A
self-contained reporting task floods main context with intermediate work
(file reads, grep results) that should have stayed isolated in a
subagent.

**Fix:** Convert to a subagent at `.claude/agents/<name>.md`, or keep as a
skill but add `context: fork` + `agent: <type>` to run the skill content
as an isolated subagent task. See `skill-vs-subagent.md`.

## 9. Over-broad `allowed-tools`

**Signals:** `allowed-tools: Read Write Edit Bash` on a skill that only
reads files. `allowed-tools: Bash` without command-prefix scoping
(`Bash(git status *)`, `Bash(npm test *)`).

**Why it bites:** `allowed-tools` pre-approves tools without per-use
prompts while the skill is active. Granting more than the skill actually
needs defeats the permission system's safety value — a misfiring skill
can do damage the user would normally have caught.

**Fix:** Grant the minimum. Use scoped Bash rules:
`Bash(git status *), Bash(git diff *)` not raw `Bash`. Audit by reading
the body and listing every tool call that appears — that's the floor.

## 10. Side-effect skill auto-invocable

**Signals:** Skill performs destructive or externally-visible actions
(deploy, commit, push, send Slack, open PR, post comment) without
`disable-model-invocation: true` in the frontmatter.

**Why it bites:** Claude can auto-fire the skill on prompts that *sound*
related, triggering side effects the user didn't authorize. "My code
looks done" could trigger `/deploy`.

**Fix:** Add `disable-model-invocation: true`. Users still invoke
manually with `/skill-name`; Claude cannot auto-fire. Pair with scoped
`allowed-tools` for defense in depth.
