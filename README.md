# repo-gardener

Context-tree-aware PR and issue maintenance agent. Babysit your repo's open work items with autonomous decision-making.

## What it does

repo-gardener runs on a schedule (or manually) and:

1. **Scans** open PRs and issues
2. **Triages** — direct fix (CI errors, lint, typos) vs context-informed (design decisions, architecture)
3. **Fixes** direct issues immediately, reads your context tree for product-level decisions
4. **Reverts** its own fixes if CI still fails after 2 attempts
5. **Identifies tree gaps** — every time it can't decide, it tells you what's missing from your context tree

Over time, your context tree gets more complete and the agent needs less human intervention.

## Requirements

- [Claude Code](https://claude.ai/code) CLI
- [GitHub CLI](https://cli.github.com/) (`gh`) authenticated
- A [context tree](https://github.com/unispark-inc/first-tree-cli) set up via `first-tree init`

## Usage

### As a slash command

Copy `.claude/commands/gardener.md` into your repo's `.claude/commands/` directory, then:

```
/gardener
```

### With Claude Code schedule (runs in the cloud)

```
/schedule every hour /gardener
```

### With /loop (runs locally)

```
/loop 1h /gardener
```

## How it works

```
Scan PRs + issues
  → Nothing open? Exit.
  → Check prior gardener state (via PR comments)
  → Prioritize: CI fail > review comments > bugs > features
  → Pull context tree (only if needed)
  → For each item:
      → Acquire lock (prevent concurrent runs)
      → Triage: direct fix or context-informed?
      → Direct: fix + push + move on
      → Context-informed: read tree → decide or hand to human
      → Don't wait for CI — check results next run
  → Log results + tree gaps
```

## State machine

repo-gardener tracks its state via PR/issue comments:

| State | Meaning |
|-------|---------|
| `gardener:in-progress` | Currently being processed (lock) |
| `gardener:pending` | Fix pushed, awaiting CI |
| `gardener:pass` | Fix landed, CI green |
| `gardener:reverted` | Fix failed, reverted |
| `gardener:failed` | 2 attempts exhausted, needs human |

## Context tree integration

repo-gardener reads a URL from your `CLAUDE.md` or `AGENT.md` pointing to your context tree repo. If no tree is found, it stops and asks you to run `first-tree init`.

For direct fixes (type errors, lint, build), no tree is needed. For anything that requires product judgment, the agent reads the tree and either:

- **Decides autonomously** if the tree has enough context (cites the tree node)
- **Hands back to human** if the tree lacks guidance, and suggests what node to add

Every handback is a signal. Fill the gap, and next time the agent handles it alone.

## License

MIT
