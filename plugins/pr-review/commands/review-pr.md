---
description: "Comprehensive PR review using specialized agents with triage and immediate fixes"
argument-hint: "[review-aspects]"
allowed-tools: ["Bash", "Glob", "Grep", "Read", "Task"]
---

# Comprehensive PR Review

Run a comprehensive pull request review using multiple specialized agents, each focusing on a different aspect of code quality. After agents finish, **always** run the triager to filter false positives, then act on the cleaned findings according to the rules below.

**Review Aspects (optional):** "$ARGUMENTS"

## Review Workflow

### 1. Determine Review Scope

- Run `git status` and `git diff --name-only` to identify changed files
- Check if a PR already exists: `gh pr view` (don't fail if it doesn't)
- Parse `$ARGUMENTS` to see if the user requested specific aspects
- Default: run all applicable reviews

### 2. Available Review Aspects

| Aspect | Agent | What it covers |
|--------|-------|----------------|
| `code` | code-reviewer | CLAUDE.md compliance, bugs, general quality |
| `tests` | pr-test-analyzer | Test coverage and quality |
| `comments` | comment-analyzer | Comment accuracy and rot |
| `errors` | silent-failure-hunter | Error handling, silent failures |
| `types` | type-design-analyzer | Type design and invariants |
| `ux` | ux-reviewer | User-facing UI/UX, accessibility, states, copy |
| `perf` | performance-reviewer | N+1, indexes, renders, bundle, hot paths |
| `security` | security-reviewer | AuthN/AuthZ, injection, secrets, crypto |
| `simplify` | code-simplifier | Final polish — only after issues are fixed |
| `all` | (all of the above) | Default |

### 3. Determine Applicable Reviews from Diff

Don't run every agent on every PR — pick the ones whose scope intersects the diff. Use this routing:

- **code-reviewer**: always applicable
- **pr-test-analyzer**: if test files changed OR if production code changed without corresponding tests
- **comment-analyzer**: if comments/docstrings/docs added or modified
- **silent-failure-hunter**: if `try/catch`, `Result`, `?`, error handling, or new failure paths touched
- **type-design-analyzer**: if new types/interfaces/structs/enums added or modified
- **ux-reviewer**: if user-facing files touched (`.tsx`, `.jsx`, `.vue`, `.svelte`, `.astro`, `.html`, `.css`, templates, layouts)
- **performance-reviewer**: if data fetching, list rendering, query/SQL, request handling, hot-path code, or new dependencies added
- **security-reviewer**: if anything touches auth, user input, external APIs, file I/O, persistence, secrets, or crypto
- **code-simplifier**: only run **after** other agents pass and the user is ready for polish

If `$ARGUMENTS` specifies aspects explicitly (e.g. `review-pr ux security`), run only those.

### 4. Launch Review Agents in Parallel

Launch all applicable agents in **parallel** via the Task tool — single message with multiple Task tool calls. This is faster and the triager handles overlap. Do not run sequentially unless the user explicitly asks for it.

Each agent receives the same context: the diff scope, file list, and any additional instructions from `$ARGUMENTS`.

### 5. Triage — MANDATORY

After all reviewer agents return, **always** launch `findings-triager` via the Task tool, passing it the combined raw findings from every agent. Do not skip this step, even if only one agent ran.

The triager will:
- Verify each finding against the actual code
- Reject false positives, duplicates, and noise
- Separate **introduced by this PR** from **pre-existing** issues
- Return a clean, prioritized report

### 6. Present the Triaged Report

Show the triager's report to the user with this structure:

```markdown
# PR Review

## Actionable (introduced by this PR)
- Critical: ...
- Important: ...

## Pre-existing (in touched files, not caused by this PR)
- ...

## Rejected (with reasoning)
- ... (collapsed by default — just count, expand on request)
```

### 7. Take Action — Rules

After presenting the report, **act on findings according to these rules without asking** for the actionable section:

**Actionable findings (introduced by this PR)**:
- **Fix immediately.** Do not ask "should I fix this?" — that's why the user ran the review.
- Apply fixes in this order: Critical → Important.
- For each fix, briefly state what you're changing and why, then make the edit.
- After all fixes, suggest re-running `/pr-review:review-pr` on the affected aspects to verify.

**Pre-existing findings (in touched files)**:
- **Stop and ask the user.** Pre-existing issues are a scope decision, not a default-yes.
- Present them as a short list and ask:
  > "Found N pre-existing issues in files this PR touches. Would you like me to:
  > 1. Fix them now (folded into this PR)
  > 2. Fix them in a separate commit on this branch
  > 3. Open a follow-up task in the tracker
  > 4. Leave them alone"
- Wait for the user's choice before doing anything.

**Rejected findings**:
- Don't act on them. Show the count only. If the user disagrees with a rejection, they will say so.

## Usage Examples

**Full review (default):**
```
/pr-review:review-pr
```

**Specific aspects:**
```
/pr-review:review-pr ux security
/pr-review:review-pr perf
/pr-review:review-pr code tests
```

**Final polish pass (after fixes):**
```
/pr-review:review-pr simplify
```

## Workflow Integration

**Before committing:**
```
1. Write code
2. /pr-review:review-pr code errors
3. Triager output → fix actionable, decide on pre-existing
4. Commit
```

**Before creating a PR:**
```
1. Stage all changes
2. /pr-review:review-pr
3. Triager output → fix actionable, decide on pre-existing
4. Re-run aspects that had findings to verify
5. Create PR
```

**After PR feedback from a human reviewer:**
```
1. Make requested changes
2. /pr-review:review-pr <relevant aspects>
3. Verify
4. Push
```

## Notes

- Agents run in parallel for speed; the triager always runs after them.
- The triager is what makes review output trustworthy — never skip it.
- Actionable fixes happen automatically; pre-existing fixes require user consent.
- All agents available via the `Task` tool with their `pr-review:` prefix.
