---
name: reviewing-claude-skills
description: >
  Audits Claude Code skills (SKILL.md + reference/) against Anthropic's skill
  authoring best practices. Covers progressive disclosure rules,
  description-trigger design, frontmatter pitfalls, the skill-vs-subagent
  boundary, and a 10-item anti-pattern scan. Invoke when reviewing, auditing,
  critiquing, polishing, or improving an existing SKILL.md or its reference
  files, when deciding whether work should be a skill or a subagent, or when a
  skill isn't triggering as expected. Skip for authoring brand-new skills from
  scratch (different shape) and for reviewing Agent SDK / Claude API code
  unrelated to skills.
---

# reviewing-claude-skills

## Design principles

- **All tiers are agent prompt.** Both SKILL.md and `reference/*.md` exist to
  serve the agent. Human-reader content (credits, sources, design rationale,
  changelog) doesn't belong in either tier; git history and an adjacent README
  preserve provenance. Default to deletion.
- **Progressive disclosure earns its keep.** Three tiers: name+description
  always loaded, SKILL.md body on relevance, `reference/*.md` on demand. Every
  line at every tier should pass "does the agent need this to do its job?"
- **State, don't narrate.** SKILL.md body stays in context across turns once
  invoked. Cut meta-narration ("this file holds the high-frequency content"),
  redundant intros, and prose explaining the file's own organization.
- **Description does triple duty.** WHAT the skill does, WHEN to invoke
  (trigger phrases, file patterns, symptoms), and NEGATIVE triggers (when to
  skip). Combined `description` + `when_to_use` is truncated at 1,536 chars in
  the skill listing — lead with the key use case.
- **Skills are reference content; subagents are isolated tasks.** If the work
  produces verbose output the main agent discards, runs autonomously, and
  returns a summary, it's a subagent. If it's rules the main agent applies
  during iterative work, it's a skill. See `reference/skill-vs-subagent.md`.

## Anti-pattern scan

Each item has signals and concrete fix in `reference/anti-patterns.md`.

1. **Length.** SKILL.md > 500 lines, or reference file > 100 lines without a
   `## Contents` TOC at the top.
2. **Weak description.** Missing trigger phrases (file patterns, symptoms,
   task verbs) or missing negative triggers ("skip for X").
3. **Description overflow.** Combined `description` + `when_to_use` > 1,536
   chars; the tail (often where negative triggers live) gets truncated.
4. **Meta-narration in SKILL.md body.** Prose describing the file itself
   instead of instructing the agent.
5. **Human-reader content.** `## Sources`, `## Credits`, `## Changelog`,
   `## Acknowledgments` sections in SKILL.md or `reference/`.
6. **Orphaned reference files.** `reference/foo.md` not linked from SKILL.md
   is invisible to Claude.
7. **Description-body mismatch.** Triggers in the description don't reflect
   what the body actually covers. Skill fires wrong or fails to fire.
8. **Should-be-a-subagent.** Skill describes a self-contained task that
   produces a discarded report rather than reference content for iterative
   work.
9. **Over-broad `allowed-tools`.** Grants Write/Edit/Bash when only Read/Grep
   are used, or raw `Bash` without command-prefix scoping.
10. **Side-effect skill auto-invocable.** Skill with destructive or
    externally-visible effects (deploy, commit, push, send) lacks
    `disable-model-invocation: true`.

## Audit checklist

Run in order on every skill:

1. **Frontmatter.** `name` kebab-case ≤64 chars; `description` length sane;
   only fields the skill actually uses (no clutter).
2. **Description triggers.** Read just the description. Would Claude know when
   to fire it? Try common phrasings the user would type. Are negative triggers
   present and concrete?
3. **Body conciseness.** Scan SKILL.md for meta-narration, credits, redundant
   intros. Every section should be instructional.
4. **Length.** SKILL.md ≤ 500 lines; each `reference/*.md` ≤ 500 lines; TOC
   for any reference file > 100 lines.
5. **Reference linkage.** Every `reference/*.md` referenced from SKILL.md with
   a one-line summary of contents.
6. **Tier discipline.** Anything in SKILL.md not applied every invocation
   belongs in `reference/`. Anything in either tier that doesn't serve the
   agent belongs in a sibling README or git history.
7. **Primitive choice.** Confirm skill is the right primitive vs subagent.
   See `reference/skill-vs-subagent.md`.

## Quick reference

| Symptom                                  | Likely cause                                  |
|------------------------------------------|-----------------------------------------------|
| Skill never auto-triggers                | Description missing trigger phrases           |
| Skill fires on irrelevant prompts        | Missing negative triggers; vague description  |
| Description truncated in `/skills` menu  | Combined description > 1,536 chars            |
| Multiple skills' descriptions cut short  | Listing budget overflow (run `/doctor`)       |
| Skill stops influencing after first turn | Auto-compaction dropped older skill content   |
| Self-contained task with discarded report| Convert to subagent or add `context: fork`    |
| Always-on knowledge across all sessions  | Move to CLAUDE.md instead of a skill          |
| Auto-fires too broadly                   | Add `paths:` glob to scope auto-activation    |
| Destructive side effects                 | Add `disable-model-invocation: true`          |

## Deeper references

- **`reference/anti-patterns.md`**: the 10 anti-patterns above with signals,
  why each bites, and concrete fix.
- **`reference/skill-vs-subagent.md`**: decision matrix, hybrid patterns
  (`context: fork`, subagent `skills:` field), and worked examples.
