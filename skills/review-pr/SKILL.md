---
name: review-pr
description: Use when reviewing a GitHub pull request - dispatches 4 specialized reviewers in parallel (structure, implementation, consistency, tests) and synthesizes findings. Auto-detects PR from current branch if no argument given.
argument-hint: <pr-number-or-url> (optional - auto-detects from current branch)
allowed-tools: Bash(gh pr view *), Bash(gh pr diff *), Bash(gh pr checks *), Bash(gh pr list *), Bash(gh api *), Bash(git branch *), Bash(git rev-parse *), Read, Grep, Glob, Agent
---

# Review GitHub Pull Request

Review PR: **$ARGUMENTS**

**Read-only review.** Do NOT post comments, approve, or request changes on GitHub. All feedback is presented in the terminal only.

## Step 1 — Resolve PR target

- If `$ARGUMENTS` is provided (number or URL), use it directly.
- If `$ARGUMENTS` is empty, auto-detect the PR for the current branch:
  ```bash
  gh pr view --json number --jq .number
  ```
  If this fails (no PR for current branch), stop and ask the user for a PR number.

For the rest of this skill, refer to the resolved PR as `<PR>`.

## Step 2 — Gather PR context

Run these commands in parallel (single message, multiple Bash calls):

- `gh pr view <PR>` — metadata: title, author, body, base branch, labels
- `gh pr diff <PR>` — full diff
- `gh pr view <PR> --comments` — existing review conversation
- `gh pr checks <PR>` — CI status
- `gh pr view <PR> --json files --jq '.files[].path'` — list of changed files

Briefly summarize for the user: PR title, author, file count, CI status.

## Step 3 — Dispatch 4 reviewers in parallel

**CRITICAL:** Send a single message with 4 Agent tool calls. Do NOT dispatch sequentially — that defeats the purpose of parallel review.

Each subagent gets the diff plus a specific lens. Briefings below are self-contained — copy them into the Agent prompt with the diff and PR context filled in.

All 4 reviewers must output findings in this exact format so synthesis is consistent:

```
- **Severity:** Block | Major | Minor | Nit
- **File:line(s):** path/to/file.ts:42-45
- **Issue:** [what's wrong]
- **Suggestion:** [what would be better]
- [reviewer-specific field, see below]
```

**Severity rubric** (every reviewer uses this):
- **Block** — fundamentally wrong; should not merge as-is
- **Major** — significant issue worth fixing before merge
- **Minor** — worth addressing but not blocking
- **Nit** — cosmetic / preference

If a reviewer finds nothing, they return "No findings."

---

### Reviewer 1: Structure (general-purpose agent)

**Prompt template:**

```
You are reviewing PR #<PR> from a STRUCTURAL perspective only. Your lens is Sean's personal engineering philosophy.

First, read `~/.claude/CLAUDE.md` to load the full philosophy. Apply principles 1-9 (Simplicity, Form, Pause-on-add, If-statements-suck, Edge cases, Abstractions, Root cause, Engineer-first, Pragmatism) to the diff below.

PR context:
- Title: <title>
- Description: <body>
- Base branch: <base>
- Files changed: <file list>

Diff:
<full diff>

Look specifically for:
1. **Simplicity & restraint** — Does this add more complexity than needed? Could existing structure absorb the new responsibility?
2. **Form** — Is the structure right? Does it look aesthetically coherent, or does it feel grafted on?
3. **Conditionals** — Scattered low-level ifs? Distinguish dispatching (should be polymorphism/lookup) vs guards (fine) vs business rules vs feature flags (must be at boundaries, not deep).
4. **Edge cases** — Are edge cases self-inflicted by the design? 3+ special cases on the same concept means the concept is modeled wrong.
5. **Abstractions** — Premature abstraction (1 caller)? Same-shape-different-concept abstraction? Missing abstraction where concepts are genuinely shared?
6. **Pause-on-add** — Are additions deliberate, or did the author add things existing structure could have absorbed?

Output: markdown bulleted findings in the standard format. Add a **Principle:** field citing the specific philosophy principle (e.g., "Principle #4 — If statements suck").

Return findings only. No preamble.
```

---

### Reviewer 2: Implementation (general-purpose agent)

**Prompt template:**

```
You are reviewing PR #<PR> for IMPLEMENTATION CORRECTNESS only.

PR context:
- Title: <title>
- Description: <body>
- Files changed: <file list>

Diff:
<full diff>

You may read adjacent files (callers, related types, definitions) to understand context. Use Read/Grep as needed.

Look for:
1. **Correctness** — Does the code actually do what the PR description / function name claims?
2. **Bug hunting** — Off-by-one, null/undefined handling, race conditions, type confusion, integer overflow.
3. **Error handling** — Are error paths covered? Is it defensive overkill or under-defensive? (Default: validate at boundaries, trust internal code.)
4. **Domain edge cases** — Timezone bugs, empty inputs, boundary values, encoding issues. (These are the *irreducible* edge cases — different from design-imposed edge cases.)
5. **Naming** — Are names accurate, or misleading in a way that masks the actual behavior?

Output: markdown bulleted findings in the standard format. No principle field needed.

Return findings only. No preamble.
```

---

### Reviewer 3: Consistency (Explore agent)

**Prompt template:**

```
You are reviewing PR #<PR> for CONSISTENCY with the existing codebase.

PR context:
- Title: <title>
- Files changed: <file list>

Diff:
<full diff>

Your job: identify where this PR diverges from existing patterns. Use Grep/Glob extensively to find similar code elsewhere in the repo.

Look for:
1. **Naming conventions** — Does this match how similar concepts are named elsewhere?
2. **Architectural patterns** — File organization, module boundaries, layering — does this follow the codebase's existing structure?
3. **Library/utility usage** — Is this reinventing something that already exists in the repo (a util, helper, type, etc.)?
4. **Idioms** — Are there established patterns used elsewhere that would be more natural here?
5. **Type/interface patterns** — Does this match how similar data is modeled elsewhere?

For each finding, include a **Reference:** field pointing to where the existing pattern lives (file:line). This is non-negotiable — a consistency claim without a reference is useless.

Output: markdown bulleted findings in the standard format, with the extra **Reference:** field.

Return findings only. No preamble.

Search breadth: medium-to-thorough. Don't just check the obvious neighbors; look across the codebase for established patterns.
```

---

### Reviewer 4: Tests (general-purpose agent)

**Prompt template:**

```
You are reviewing PR #<PR> for TEST COVERAGE and QUALITY.

PR context:
- Title: <title>
- Description: <body>
- Files changed: <file list>

Diff:
<full diff>

- Coverage goal: **catch regressions, not chase numbers**
- Test branching logic, edge cases, prior bug sites, anything where silent regression would be expensive
- Don't test: framework code, trivial getters/setters, pure passthroughs, all-mocked tests that verify nothing, brittle UI snapshots, behavior nobody cares about
- Bug fixes should have a regression test that fails without the fix

You may read adjacent test files to understand existing patterns. Use Read/Grep as needed.

Look for:
1. **Coverage gaps** — Is new logic actually tested? What branches are uncovered?
2. **Test quality** — Are tests testing the right things, or just locking in implementation details?
3. **Over-mocking** — Tests with everything mocked that verify nothing real?
4. **Regression tests** — If this is a bug fix, is there a test that would have caught the bug? Would the new test fail without the fix?
5. **Brittleness** — Snapshot tests, timing-dependent tests, tests that will fail for cosmetic reasons?
6. **Wasted tests** — Tests that shouldn't exist (testing framework code, trivial accessors, all-mocked nothing-tests).

Output: markdown bulleted findings in the standard format.

Return findings only. No preamble.
```

---

## Step 4 — Synthesize

Once all 4 reviewers return, collect findings and produce a structured review. Do this work in the main agent, not in another subagent.

1. **Deduplicate** — if multiple reviewers flagged the same line/issue, merge into one finding and mark **High confidence (caught by N reviewers)**.
2. **Group by severity** — Block, Major, Minor, Nit. Within each section, order by file.
3. **Verdict** — based on the findings:
   - Any Block items → **Request changes**
   - Any Major items → **Approve with comments** (or Request changes if there are several)
   - Only Minor / Nit → **Approve**

## Step 5 — Output

Print to terminal as markdown. Structure:

```
# PR Review: <title>

**Author:** <author>
**Base:** <base branch>
**Files changed:** <count>
**CI:** <status>

## Verdict: <APPROVE | APPROVE WITH COMMENTS | REQUEST CHANGES>

<one-sentence rationale>

---

## Block

<list, or "None">

## Major

<list, or "None">

## Minor

<list, or "None">

## Nits

<list, or "None">

---

## Cross-cutting observations

<any patterns that emerged across reviewers, or "None">
```

For each finding, include all fields the reviewer provided (Severity, File:line, Issue, Suggestion, Principle/Reference where applicable). Mark high-confidence items (multiple reviewers caught) with **[High confidence — N reviewers]**.

## Notes

- **Do NOT post anything to GitHub.** This is terminal-only.
- **Don't editorialize beyond the findings.** The reviewers did the analysis; synthesis assembles their work, it doesn't add new opinions.
- If a reviewer returned "No findings," that's a signal — note in the output ("Structure reviewer found no issues") rather than omitting silently.
