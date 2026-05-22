# Skill vs subagent: choosing the primitive

## Contents

- The official guidance
- Decision criteria
- Decision table
- Hybrid patterns
- Worked examples

## The official guidance

From the Claude Code subagents docs, "Choose between subagents and main
conversation":

> Use **subagents** when:
> - The task produces verbose output you don't need in your main context
> - You want to enforce specific tool restrictions or permissions
> - The work is self-contained and can return a summary
>
> Consider **Skills** instead when you want reusable prompts or workflows
> that run in the main conversation context rather than isolated subagent
> context.

And on subagent context isolation:

> Each subagent starts with a fresh, isolated context window. It does not
> see your conversation history, the skills you've already invoked, or the
> files Claude has already read.

The fresh-context cost is the central tradeoff. A subagent gains
isolation but loses everything the main thread already knows. A skill
keeps surrounding context but pays the recurring-token cost of its body.

## Decision criteria

Run through these. Lean toward whichever side wins the majority.

| Question                                                | Skill | Subagent |
|---------------------------------------------------------|:-----:|:--------:|
| Output applied in-place during iterative work?          |   ✓   |          |
| Output is a summary/report that's read once and dropped?|       |    ✓     |
| Needs main-thread context (open files, conversation)?   |   ✓   |          |
| Can start from scratch with just a task prompt?         |       |    ✓     |
| Produces verbose intermediate output (logs, dumps)?     |       |    ✓     |
| Reference content the agent applies as it writes code?  |   ✓   |          |
| Needs tool restrictions (read-only, no network)?        |       |    ✓     |
| Runs autonomously without back-and-forth?               |       |    ✓     |
| Pattern is "here's how to do X" (procedural knowledge)? |   ✓   |          |
| Pattern is "go do X and report back" (discrete task)?   |       |    ✓     |

## Hybrid patterns

These exist because the boundary isn't binary. Both keep the skill as the
source of truth for procedural knowledge.

### Skill that runs in isolation: `context: fork`

```yaml
---
name: deep-research
description: Research a topic thoroughly
context: fork
agent: Explore
---

Research $ARGUMENTS thoroughly:
1. Find relevant files
2. Read and analyze the code
3. Summarize findings with file references
```

The skill content becomes the subagent's task prompt. Use when:

- The skill content is a discrete task, not pure reference
- You want isolation (don't pollute main context with intermediate work)
- The skill stays the source of truth, but execution is offloaded

**Caveat:** `context: fork` only works for skills with explicit task
instructions. Reference-only skills ("use these conventions") have no
actionable prompt and return nothing useful.

### Subagent that preloads a skill: `skills:` field

```yaml
---
name: pytest-author
description: Write pytest tests for a given module
skills:
  - testing-with-pytest
---

Write tests for $ARGUMENTS following the preloaded conventions.
```

The full content of `testing-with-pytest` is injected into the
subagent's context at startup. Use when:

- You have a specialist subagent (isolated, autonomous worker)
- The subagent should follow rules defined in an existing skill
- You want the skill to stay shared with the main thread *and* available
  to the subagent

## Worked examples

**`testing-with-pytest`** → skill. Reference patterns the main agent
applies while writing tests. Needs to see existing tests, conftest.py,
source under test. No discardable verbose output.

**`reviewing-bash`** → skill. Rules applied during code review with
citations of authorities (Wooledge, ShellCheck). Output is inline
feedback, not a discardable report.

**`/code-review`** (bundled) → skill, despite report-shaped output. Runs
in the main thread so the user can immediately ask follow-ups, jump into
the diff, or apply fixes without re-spawning a subagent.

**`/run`, `/verify`** (bundled) → skills. Launch and drive the app; the
result (success/failure, screenshot) is small and needed by the main
thread for the next decision.

**Explore** (built-in subagent) → subagent. Produces verbose
intermediate output (file contents, grep results); only a summary
returns. Read-only tool restrictions enforce safety.

**Plan** (built-in subagent) → subagent. Research-heavy; isolates
exploration from the main planning context.

**Custom code-reviewer specialist** → subagent. Reads a PR diff, returns
structured findings. Read-only, autonomous, summary-shaped output. Can
be paired with a `reviewing-bash` or `testing-with-pytest` preload via
the `skills:` field.

**Edge case — "audit every skill in this workspace"**: verbose
intermediate work (reading every SKILL.md and reference file), but the
final output is structured findings the user will act on. Either a
skill with `context: fork` (keeps audit rules shareable) or a subagent
that preloads `reviewing-claude-skills` works. Default to the skill +
fork pattern.
