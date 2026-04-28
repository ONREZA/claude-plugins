---
description: "Comprehensive PR review using specialized agents with file-based exchange and triage"
argument-hint: "[review-aspects]"
allowed-tools: ["Bash", "Glob", "Grep", "Read", "Task", "AskUserQuestion"]
---

# Comprehensive PR Review

Run a comprehensive pull request review using multiple specialized agents, each focusing on a different aspect of code quality. Agents write findings to a shared run directory; the triager reads, verifies, dedupes, and produces the final report.

**Review Aspects (optional):** "$ARGUMENTS"

## Run Directory — File-Based Exchange

Each invocation creates a run directory grouping all artifacts:

```
.claude/tmp/pr-review/run-<timestamp>/
└── findings/
    ├── code-reviewer.md
    ├── security-reviewer.md
    ├── performance-reviewer.md
    ├── ...
    └── triaged.md          # written by findings-triager
```

Reviewers write to `<agent-name>.md`. The orchestrator does **not** relay finding text between agents — it passes only `FINDINGS_PATH`. The triager reads every file in the directory itself.

## Agent Models

| Agent | Model | Notes |
|-------|-------|-------|
| `code-reviewer` | opus | |
| `security-reviewer` | opus | |
| `findings-triager` | opus | |
| `performance-reviewer` | sonnet | upgrade to opus for: complex SQL plans, memory leak analysis, algorithmic optimization |
| `type-design-analyzer` | sonnet | |
| `silent-failure-hunter` | sonnet | |
| `pr-test-analyzer` | sonnet | |
| `comment-analyzer` | sonnet | |
| `ux-reviewer` | sonnet | |
| `code-simplifier` | sonnet | post-fix polish, separate workflow |

## Workflow

### 1. Determine review scope

- `git status` and `git diff --name-only` to find changed files
- `gh pr view` to detect an existing PR (do not fail if absent)
- Parse `$ARGUMENTS` for explicit aspects

### 2. Set up the run directory (once, before any agents)

```bash
RUN_ID="run-$(date +%Y%m%dT%H%M%S)"
RUN_DIR=".claude/tmp/pr-review/$RUN_ID"
FINDINGS_PATH="$(pwd)/$RUN_DIR/findings"
mkdir -p "$FINDINGS_PATH"

# self-ignore so user's repo never tracks run artifacts, even if .claude/ is not gitignored
if [ ! -f .claude/tmp/.gitignore ]; then
  printf '*\n' > .claude/tmp/.gitignore
fi

# clean runs older than 7 days, ignore failures
find .claude/tmp/pr-review -maxdepth 1 -type d -name 'run-*' -mtime +7 -exec rm -rf {} + 2>/dev/null || true
```

`FINDINGS_PATH` must be **absolute** — agents need it to be unambiguous regardless of their cwd.

### 3. Available aspects

| Aspect | Agent | Covers |
|--------|-------|--------|
| `code` | code-reviewer | CLAUDE.md compliance, bugs, general quality |
| `tests` | pr-test-analyzer | Test coverage and quality |
| `comments` | comment-analyzer | Comment accuracy and rot |
| `errors` | silent-failure-hunter | Error handling, silent failures |
| `types` | type-design-analyzer | Type design and invariants |
| `ux` | ux-reviewer | User-facing UI/UX, a11y, states, copy |
| `perf` | performance-reviewer | N+1, indexes, renders, bundle |
| `security` | security-reviewer | AuthN/Z, injection, secrets, crypto |
| `simplify` | code-simplifier | Final polish — separate from review pipeline |
| `all` | (everything except `simplify`) | Default |

### 4. Pick applicable agents from the diff

- **code-reviewer** — always
- **pr-test-analyzer** — tests changed OR production code changed without corresponding tests
- **comment-analyzer** — comments / docstrings / docs added or modified
- **silent-failure-hunter** — try/catch, Result, ?, error handlers, new failure paths
- **type-design-analyzer** — new types/interfaces/structs/enums or modifications to existing ones
- **ux-reviewer** — user-facing files (`.tsx`, `.jsx`, `.vue`, `.svelte`, `.astro`, `.html`, `.css`, templates, layouts)
- **performance-reviewer** — data fetching, list rendering, queries, request handling, hot-path code, new dependencies
- **security-reviewer** — auth, user input, external APIs, file I/O, persistence, secrets, crypto
- **code-simplifier** — only when explicitly requested or after fixes are applied

If `$ARGUMENTS` lists explicit aspects, run only those.

### 5. Confirm the plan

Show the plan to the user and ask via AskUserQuestion:

```markdown
## PR Review Plan

| Agent | Model | Why |
|-------|-------|-----|
| code-reviewer | opus | always |
| security-reviewer | opus | auth/input handling detected in diff |
| ... | ... | ... |

Run directory: `.claude/tmp/pr-review/<run-id>/`
```

**Question:** "Confirm review plan"
- "Run as proposed" (Recommended)
- "Modify agents" — multi-select with applicable agents pre-checked, allows per-agent model override
- "Upgrade all to opus" — for maximum depth
- "Downgrade to sonnet" — for faster/cheaper review
- "Cancel"

**Consider upgrading performance-reviewer to opus** if the diff shows complex SQL plans (multi-join, index optimization), memory leak analysis (deep async chains, closures), or algorithmic optimization.

### 6. Launch reviewers in parallel

Single message with one Task call per selected agent. Each agent receives:

- `FINDINGS_PATH` — absolute directory from step 2
- `MODEL` — chosen model
- Diff scope (changed files, commit range)
- Any user-supplied focus from `$ARGUMENTS`

Each agent writes its file to `$FINDINGS_PATH/<agent-name>.md` and returns a one-line confirmation. **Do not** pass finding text between agents.

### 7. Triage — mandatory, always

After every reviewer task returns, launch `findings-triager` via Task with:

- `FINDINGS_PATH` — same directory
- `PR_RANGE` — commit range or branch info, if known

Run the triager **even if only one reviewer ran** or if some reviewers reported `no-findings`. The triager reads every `*.md` in `$FINDINGS_PATH` (excluding `triaged.md`), verifies each finding (reading the code, reproducing in `/tmp/` when needed), dedupes, categorizes, and writes `$FINDINGS_PATH/triaged.md`.

### 8. Present the triaged report

Read `$FINDINGS_PATH/triaged.md` and present it. Do **not** paraphrase. Preserve:

- The structure (Introduced / Pre-existing / Rejected)
- The per-finding axes (`scope` · `risk` · `related` · `action`)
- The source agents list
- Suggested PR/issue titles for `separate-pr` / `track` items

### 9. Act on findings — by `action` label

For each entry, follow its action:

- **`fix-now`** — fix immediately. State what you are changing in one line, then make the edit. Apply Critical → Important → Minor.
- **`fix-in-pr`** — ask once: "Fold these N findings into this PR or leave them?". Do not silently apply.
- **`separate-pr`** — never auto-apply. Offer:
  1. Open a follow-up branch + PR using the suggested title
  2. Write a tracker issue
  3. Leave alone
- **`track`** — propose the suggested issue/task title to the user; do not implement.

For **pre-existing** findings: never auto-fix, even when the action is `fix-now`-shaped. Ask the user — pre-existing fixes are a scope decision.

After all `fix-now` items are applied, suggest re-running the affected aspects to verify.

## Usage

```
/pr-review:review-pr                  # full review
/pr-review:review-pr ux security      # specific aspects
/pr-review:review-pr perf
/pr-review:review-pr simplify         # post-fix polish only (separate flow)
```

## Workflow integration

**Before committing:**
```
1. Write code
2. /pr-review:review-pr code errors
3. Read triaged.md → apply fix-now, decide on rest
4. Commit
```

**Before opening a PR:**
```
1. Stage all changes
2. /pr-review:review-pr
3. Read triaged.md → apply fix-now, decide on rest
4. Re-run affected aspects to verify
5. gh pr create
```

**After human PR feedback:**
```
1. Make requested changes
2. /pr-review:review-pr <relevant aspects>
3. Verify
4. Push
```

## Notes

- Agents run in parallel. The triager always runs after them.
- Reviewers must not return findings in their response — they write to `$FINDINGS_PATH/<agent>.md`. The orchestrator only relays a path.
- The triager is the only step that produces user-facing output. Never skip it.
- `fix-now` applies automatically; `fix-in-pr` / `separate-pr` / `track` require user consent.
- Old runs are kept for 7 days, then auto-cleaned.
- `code-simplifier` is **not** part of the standard review pipeline — it runs separately on demand after fixes are in place.
