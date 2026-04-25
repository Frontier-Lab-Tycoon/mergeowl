# mergeowl Spec

> Working draft for product and implementation discussions.

## One-line summary

mergeowl is a playful AI reviewer for diffs, pull requests, and patches that aims to give useful, scoped, engineer-friendly review comments instead of generic LLM chatter.

## Problem

Code review has two annoying failure modes:

1. **Humans are bandwidth-limited.** Small teams often skip deep review on routine changes, large diffs, or side projects.
2. **Current AI review is often noisy.** It tends to restate the diff, invent issues, or produce vague suggestions with poor prioritization.

The practical gap is not "an AI that understands all code perfectly." The gap is a reviewer that can:

- read a patch carefully,
- call out a small number of high-signal issues,
- explain why they matter,
- stay grounded in the actual diff,
- and be cheap/simple enough to run on every change.

## Goals

- Produce **high-signal review comments** on diffs, pull requests, and pasted patches.
- Prioritize **actionable findings**: correctness, regressions, security, edge cases, maintainability, and missing tests.
- Keep the product simple enough for a small team to iterate on prompts, evaluator datasets, and model behavior quickly.
- Make reviewer behavior legible: comments should reference concrete lines/hunks and include clear reasoning.
- Support lightweight experimentation so different prompts/models/review policies can be compared.

## Non-goals

- Fully autonomous code approval or merge authority.
- Replacing human review for architectural decisions, product judgment, or deep domain context.
- Massive enterprise workflow coverage on day one.
- Building a giant general-purpose coding agent before the core reviewer is good.

## Users

- **Indie hackers / small teams** who want a second set of eyes on every patch.
- **Maintainers** triaging incoming PRs and wanting fast initial review.
- **Researchers / agent builders** experimenting with review-oriented prompts and evaluation.
- **Developers** who want feedback on pasted diffs before opening a PR.

## Core use cases

1. A developer submits a git diff and gets a ranked list of likely issues before pushing.
2. A PR is reviewed automatically and mergeowl leaves a compact summary plus inline comments.
3. A maintainer compares multiple review prompts/models on the same patch set.
4. A user pastes a patch into a chat/UI and gets review comments without repo integration.

## Product shape

### Inputs

- Unified diffs from git
- PR patch/changed files metadata
- Optional repository context:
  - touched file contents
  - nearby code windows
  - relevant tests
  - file paths / language hints
- Optional review policy/profile:
  - be strict vs lenient
  - focus on bugs/security/perf/style
  - max comments

### Outputs

Primary output should be a compact structured review object plus a human-readable rendering.

Suggested structure:

- **summary**: 2-5 bullet overview of risk areas
- **findings**: ordered list of comments, each with:
  - severity: blocker / warning / nit
  - confidence
  - file + line/hunk reference
  - claim: what is wrong or risky
  - why: short reasoning
  - suggestion: concrete fix or check
- **uncertainties**: places where the model is not confident and wants more context
- **pass verdict**: no major issues found / needs attention

### Review behavior

mergeowl should:

- Prefer **fewer, better comments** over exhaustive noisy output.
- Stay grounded in the changed code; do not speculate beyond available evidence.
- Distinguish clearly between:
  - likely bug,
  - possible risk,
  - style suggestion.
- Avoid repeating obvious diff summaries unless they add value.
- Escalate missing tests only when the change meaningfully alters behavior.
- Admit uncertainty when the diff lacks enough context.

A good comment should look like:

- specific,
- falsifiable,
- anchored to code,
- useful to an engineer in under 30 seconds.

## System design notes

### Model strategy

Start simple.

**Baseline:**
- One strong prompting pipeline over a capable general LLM.
- Diff chunking + small amount of file context.
- Structured output schema for findings.

**Near-term extensions:**
- Two-stage review:
  1. generate candidate findings,
  2. critique/deduplicate/rank them.
- Language-specific heuristics for common bug patterns.
- Cheap prefilters to skip obviously low-risk diffs.
- Optional secondary verifier model for high-severity comments.

**Important constraint:**
This is a side-project-friendly system, so iteration speed matters more than elaborate agent stacks. A strong baseline with good evals is preferred over complicated orchestration.

### Evaluation

We should evaluate for **precision first**, then recall.

Why: a noisy reviewer gets muted quickly.

Recommended eval slices:

- real historical PRs with known review comments
- synthetic bug-injected diffs
- clean refactors / formatting-only diffs
- tests-only changes
- dependency / config changes
- security-sensitive patches

Key metrics:

- finding precision (how many comments are genuinely useful)
- severe false positive rate
- duplicate comment rate
- grounding quality (comment actually tied to diff)
- coverage of seeded bugs
- reviewer preference score from human raters

Simple rubric for each finding:

- correct / incorrect
- important / minor / irrelevant
- grounded / weakly grounded / hallucinated
- actionable / vague

### Serving / operations

Initial system should be cheap and easy to run:

- trigger on PR or manual diff submission
- compute diff chunks deterministically
- fetch minimal repo context only for touched files
- call the review model with a strict schema
- persist raw inputs/outputs for eval and regression testing

Operational priorities:

- reproducible prompts/configs
- cost per reviewed PR
- latency acceptable for developer workflow
- easy replay of old diffs through new prompts/models
- auditability of why a comment was produced

## Open questions

- Should mergeowl optimize first for **GitHub PR review** or **standalone diff/paste review**?
- How much repository context improves precision before cost/latency gets silly?
- Is a two-stage reviewer materially better than one-pass prompting for this domain?
- What is the right default comment budget: 3, 5, or 10 findings?
- Should "nit" comments exist at all in the MVP, or only warning/blocker?
- Do we want a stable JSON review format from day one for eval/UX consistency?

## Suggested MVP

A practical MVP would be:

1. Accept a unified diff.
2. Retrieve touched-file snippets only.
3. Generate up to 5 findings in a fixed schema.
4. Render them as a concise review summary.
5. Save examples for offline evaluation.

That is enough to answer the real first question:

**Can mergeowl produce comments that engineers consistently find useful, with tolerably low false positives?**
