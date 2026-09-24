# CLAUDE.md

Behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.

## 5. Working Style

Give a little bit of context. Talk in ASD-STE100 Simplified Technical English. Use the ubiquitous language from CONTEXT.md (follow CONTEXT-MAP.md to the right one if the repo has more than one). Keep comments super short and sharp and only in not straightforward places. Comment why, not how. Do not mention Slack messages, tickets, or how-tos.

In plan mode, use `/mattpocock-skills:grill-me` to create a good plan to implement.

## 6. Planning and Future Work

Store all plans and future work as GitHub issues, not as files in the repo (no `TODO.md`, no `docs/plans/`). When plan mode produces a plan the user wants to keep, create a GitHub issue for it instead of a file.

## 7. Code Review

Use `/code-review:code-review` as well for the review. Accumulate results.

## 8. Repo Best Practices

Repo-specific conventions live in `docs/best-practices.md` — read it before writing code, and add a rule to it whenever a review or a bug reveals a convention worth keeping.

## 9. Creating a Pull Request

Writing a PR description follows a strict ASD-STE100 template and a fixed creation procedure. Read `docs/pr-description.md` before drafting a PR body or running `gh pr create`.

## Agent skills

### Issue tracker

Issues live as GitHub issues in `mbobakov/prompt-rollout-operator`, driven by the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

The five default triage labels, each label string equal to its role name. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `CONTEXT.md` at the root, ADRs in `docs/adr/`. See `docs/agents/domain.md`.
