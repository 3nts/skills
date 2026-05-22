---
name: promoting-memory
description: Audits Claude Code auto-memory entries and proposes promoting project-scoped rules into the repo's CLAUDE.md as tight bullets. Invoke when the user wants to audit memory drift, prune stale entries, promote memories to CLAUDE.md, or asks "should this be in CLAUDE.md instead?". Skip when writing new memories (the harness handles that) and when authoring CLAUDE.md from scratch (this is for graduations only).
---

# Promoting memory to CLAUDE.md

## Core stance

- **Err on caution.** When ambiguous, leave in memory.
- **Minimize CLAUDE.md growth.** Each graduate becomes one tight bullet, ≤2 lines. Optional one-line example only if load-bearing. No new sections unless ≥2 entries cluster under a missing topic.
- **Per-item sign-off.** Never batch-apply. Present, gate, then write.

## Procedure

1. **Read** `MEMORY.md` and every linked entry under the project's memory dir (`$CLAUDE_CONFIG_DIR/projects/<slug>/memory/`). Read the repo's `CLAUDE.md` for what is already directive. Grep the repo for any file, function, or flag a memory names, to flag stale references.
2. **Classify** each entry:
   - **Graduate** — project rule that should bind any contributor + their agents
   - **Personal** — user preference, identity, environment, account state (stays in memory)
   - **General agent-behavior** — how the user wants agents to reason, ask, or present across projects (stays in memory)
   - **Hybrid** — project-rule half + personal half; propose a split
   - **Stale** — names files / functions / flags that no longer exist, or already duplicated in CLAUDE.md (propose delete)
3. **For each graduate** (or graduate-half of a hybrid), draft:
   - Destination section in CLAUDE.md (prefer existing over new). If no `CLAUDE.md` exists, propose creation only when ≥1 graduates would land in it.
   - Exact bullet, ≤2 lines
   - Memory action: full delete, or slimmed remainder with the personal half retained
4. **Present** as a per-entry list with classification + proposed action. For >3 graduates, use AskUserQuestion with select-many; otherwise ask in prose. Surface borderline calls explicitly so the user can override.
5. **Apply only after explicit go-ahead.** Edit CLAUDE.md, delete or shrink memory files, update `MEMORY.md` index. Confine changes to what was approved.

## Classification heuristics

- References repo paths, repo tooling, repo workflow → **Graduate** (likely)
- References user's identity, plan, environment, taste → **Personal** (stays)
- References how agents should reason, ask, or present (cross-project) → **General agent-behavior** (stays)
- Already verbatim or near-verbatim in CLAUDE.md → **Stale → delete**
- Names a file / function / flag that grep can't find → **Stale → confirm, then delete**

## Anti-patterns

- Promoting personal preferences (em-dash bans, account state, local config paths) into a team file.
- Adding "keep CLAUDE.md slim" *to* CLAUDE.md: self-referential vibes don't earn the line.
- Creating new sections for single bullets.
- Adding examples that mirror what the bullet already says.
- Batch-applying without per-item sign-off.
- Touching memories the user did not approve in this pass.

## Output shape

Per audit, present:
- **Graduates** — proposed bullet, destination section, memory action
- **Splits** — which half goes where
- **Stale** — deletion candidates with reason
- **Stays** — one-line rationale each (group by classification, don't list every one separately if many share a reason)
- **Bloat estimate** — lines added to CLAUDE.md, before/after
