# PR Review

Comprehensive PR review for Claude Code: nine specialized reviewer agents plus a triager that fact-checks their output, dedupes findings, separates pre-existing from PR-introduced issues, and recommends actions on a falsifiable axis.

The plugin is built around a single command: **`/pr-review:review-pr`**. Individual agents can also be invoked directly when needed.

## How it works

1. `/pr-review:review-pr` inspects the diff and proposes a plan (which agents, on which model). User confirms.
2. A run directory is created at `.claude/tmp/pr-review/run-<timestamp>/findings/` (and self-ignored from git).
3. All selected reviewers run **in parallel**. Each writes its findings to `<agent>.md` in the run directory. They do **not** return text — only confirmation.
4. The **findings-triager** reads every file in the directory, verifies each finding against the actual code (reproducing in `/tmp/` when needed), dedupes, categorizes, and writes `triaged.md`.
5. The orchestrator presents the triaged report and acts according to the per-finding `action` label.

The user never sees raw reviewer output — only the triaged report. The triager is what makes the review trustworthy.

## Command

```
/pr-review:review-pr                  # full review
/pr-review:review-pr ux security      # specific aspects
/pr-review:review-pr perf
/pr-review:review-pr simplify         # post-fix polish (separate flow)
```

| Aspect | Agent | Covers |
|--------|-------|--------|
| `code` | code-reviewer | CLAUDE.md compliance, bugs, general quality |
| `tests` | pr-test-analyzer | Test coverage and quality |
| `comments` | comment-analyzer | Comment accuracy and rot |
| `errors` | silent-failure-hunter | Error handling, silent failures, fallbacks |
| `types` | type-design-analyzer | Type design, invariants, encapsulation |
| `ux` | ux-reviewer | User-facing UI/UX, a11y, states, copy |
| `perf` | performance-reviewer | N+1, indexes, renders, bundle, hot paths |
| `security` | security-reviewer | AuthN/Z, injection, secrets, crypto, CSRF |
| `simplify` | code-simplifier | Final polish — separate from review pipeline |
| `all` | (everything except `simplify`) | Default |

## Agents

### Reviewers — write findings to the run directory

Each reviewer has the same output contract: it writes a structured markdown file to `$FINDINGS_PATH/<agent>.md` and returns only a one-line confirmation. Severity is uniformly `critical | important | minor`. Confidence is a 0-100 self-assessment (only findings ≥ 70 are emitted).

| Agent | Model | What it looks for |
|-------|-------|-------------------|
| **code-reviewer** | opus | Project-guideline compliance (CLAUDE.md), real bugs, code quality |
| **security-reviewer** | opus | Auth/authz gaps, injection sinks, IDOR, secret leaks, weak crypto, CSRF, webhook signature verification, insecure defaults |
| **performance-reviewer** | sonnet (upgrade to opus for SQL plans, memory leaks, algorithmic work) | N+1, missing indexes, sync I/O in async, unbounded loops, missing pagination, render thrash, bundle bloat, missing caching |
| **ux-reviewer** | sonnet | Interaction states (loading/empty/error), a11y (ARIA, focus, contrast), form ergonomics, copy quality, mobile/responsive |
| **silent-failure-hunter** | sonnet | Empty/broad catches, swallowed errors, unjustified fallbacks, retries without observability, error promotion to silent success |
| **type-design-analyzer** | sonnet | Weak encapsulation, unenforced invariants, illegal states reachable, inconsistent mutation guards |
| **pr-test-analyzer** | sonnet | Missing coverage on critical paths, edge cases, error branches; tests overfit to implementation |
| **comment-analyzer** | sonnet | Factually wrong, outdated, misleading, or redundant comments |

### Triager — fact-checks and decides actions

**`findings-triager`** (opus) reads every reviewer file, verifies each finding by Read/Grep, runs `git blame` to separate introduced-by-PR from pre-existing, builds reproductions in `/tmp/` when truth depends on runtime behavior, uses context7/WebFetch for library API claims, dedupes by location+symptom, and writes `triaged.md`.

**The triager does not curate.** It removes only:
1. Hallucinations (cited code does not exist or does not match)
2. Exact duplicates (merged into one record, all source agents listed)
3. Findings that contradict an explicit `CLAUDE.md` rule (with verbatim citation)

It does **not** drop findings for being "minor", "subjective", "stylistic", or "out of scope". `minor` is a severity, not grounds for rejection — the user decides what to fix.

### Recommendation matrix — no time estimates

Instead of hours/days/story points (which agents reliably misjudge), the triager scores each finding on three falsifiable axes:

- **scope** — `trivial` / `local` / `multi-file` / `cross-module`
- **risk** — `safe` / `moderate` / `high` (likelihood the fix breaks something)
- **relatedness** — `tight` / `near` / `tangential` (connection to the PR's goal)

These are derived mechanically from the diff and the proposed fix. The recommendation follows from a fixed matrix:

| Scope | Risk | Related | Action |
|---|---|---|---|
| trivial | safe/moderate | any | `fix-now` |
| local | safe/moderate | tight/near | `fix-now` |
| local | safe/moderate | tangential | `fix-in-pr` |
| local | high | any | `separate-pr` |
| multi-file | safe | tight | `fix-now` |
| multi-file | safe | near | `fix-in-pr` |
| multi-file | moderate/high | any | `separate-pr` |
| multi-file | any | tangential | `separate-pr` |
| cross-module | any | any | `separate-pr` |

For `separate-pr` and `track`, the triager always proposes a one-line PR/issue title.

### Action rules

- **`fix-now`** — applied automatically by the orchestrator
- **`fix-in-pr`** — user is asked once whether to fold in
- **`separate-pr`** — never auto-applied; user can open a follow-up branch / write a tracker issue / leave alone
- **`track`** — needs design first; only a suggested issue title

**Pre-existing** findings are never auto-fixed even when the action is `fix-now`-shaped — pre-existing fixes are always a scope decision, so the user is asked.

### code-simplifier — separate flow

`code-simplifier` (sonnet) is **not** part of the standard review pipeline. It runs on demand after fixes are applied (`/pr-review:review-pr simplify`). It applies edits directly rather than writing to `$FINDINGS_PATH`. It reads `CLAUDE.md` and nearby code to follow the project's actual conventions — it does not impose a stack-specific style.

## Workflow examples

**Before committing:**

```
1. write code
2. /pr-review:review-pr code errors
3. read triaged report → fix-now applied automatically; decide on rest
4. commit
```

**Before opening a PR:**

```
1. stage all changes
2. /pr-review:review-pr
3. read triaged report → fix-now applied; decide on pre-existing
4. re-run affected aspects to verify
5. gh pr create
```

**After human PR feedback:**

```
1. apply requested changes
2. /pr-review:review-pr <relevant aspects>
3. verify
4. push
```

## Run directory

```
.claude/tmp/pr-review/run-<timestamp>/
└── findings/
    ├── code-reviewer.md
    ├── security-reviewer.md
    ├── performance-reviewer.md
    ├── ...
    └── triaged.md          # written by findings-triager
```

The directory is auto-created on each run. `.claude/tmp/.gitignore` (`*`) is created automatically so artifacts never leak into commits, even if the user has not gitignored `.claude/`. Runs older than 7 days are auto-cleaned at the start of each new run.

## Installation

From the marketplace:

```
/plugin marketplace add ONREZA/claude-plugins
/plugin install pr-review@onreza-claude-plugins
```

## Direct agent invocation

For targeted use cases, agents can be invoked directly via the Task tool with their `pr-review:` prefix (e.g. `pr-review:security-reviewer`). The default workflow is the orchestrated `/pr-review:review-pr` command — direct invocation skips the triager.

## License

MIT

## Author

ONREZA — `support@onreza.ru`
