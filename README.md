# ONREZA Claude Plugins

Internal Claude Code plugin marketplace for ONREZA. Contains forks of upstream Anthropic plugins with project-specific extensions, plus custom plugins built around the ONREZA stack (Bun monorepo, Rust services, SolidJS, Elysia.js, NATS).

## Plugins

| Plugin | Description |
|--------|-------------|
| [`pr-review`](./plugins/pr-review) | Fork of Anthropic's `pr-review-toolkit`. Specialized agents for code review, comments, tests, error handling, type design, and simplification. |

## Installation

Add this marketplace to Claude Code:

```
/plugin marketplace add ONREZA/claude-plugins
```

Then install a plugin:

```
/plugin install pr-review@onreza-claude-plugins
```

## Updating from upstream

Forks of upstream plugins (like `pr-review`) are not automatically synced. To pull changes from `anthropics/claude-code`:

1. Clone upstream: `git clone --depth=1 https://github.com/anthropics/claude-code.git /tmp/upstream`
2. Diff against the corresponding directory in `plugins/`
3. Cherry-pick relevant changes manually

We do not maintain a git remote to upstream — this is intentional, since we keep only a small subset of plugins and the upstream repo is large.

## Adding a new plugin

1. Create `plugins/<name>/.claude-plugin/plugin.json` with `name`, `version`, `description`, `author`
2. Add `agents/`, `commands/`, `skills/`, or `hooks/` subdirectories as needed
3. Register the plugin in `.claude-plugin/marketplace.json`
4. Commit and push — clients pick up changes after `/plugin marketplace update onreza-claude-plugins`

See [Claude Code plugin docs](https://docs.claude.com/en/docs/claude-code/plugins) for details on plugin structure.
