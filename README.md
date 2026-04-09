# repo-gardener

**Context-aware review bot for GitHub PRs and issues.** Reviews against your project's context tree — not your code.

> 🎯 **We review context fit, not code correctness.**
> Run Greptile / CodeRabbit for code review. Run repo-gardener to catch PRs and issues that conflict with your team's prior product decisions.

---

## The problem

Your team makes product decisions every week: "we defer dark mode to v3", "we chose Paddle over Stripe", "no session tokens, use JWT rotation". These decisions live in Slack threads, design docs, and people's heads.

Three months later, a contributor opens a PR that adds dark mode. The maintainer has to:

1. Remember the decision
2. Find the rationale
3. Write a polite rejection
4. Hope the same thing doesn't happen next week

Existing OSS bots don't help here. Greptile reviews code. Dependabot bumps versions. Nobody reads your context.

**repo-gardener reads your context tree** (the knowledge base where those decisions live) and comments on PRs/issues that touch those decisions — with specific tree node references.

---

## How it compares to other OSS bots

| Bot | Lane | Output | Reads context? |
|-----|------|--------|----------------|
| **Greptile** | Code review | Inline comments, summaries, bug flags | No |
| **CodeRabbit** | Code review | Walkthroughs, inline suggestions, diagrams | No |
| **DeepSource** | Static analysis | Issue counts, autofix suggestions | No |
| **Sweep** | Issue → PR | Opens PRs from issues | No |
| **Dependabot / Renovate** | Dependency updates | Version bump PRs | No |
| **Sourcery** | Refactoring | Quality scores, refactor diffs | No |
| **🌱 repo-gardener** | **Product context fit** | **Structured comments citing tree nodes** | **Yes — reads CLAUDE.md, AGENTS.md, context tree** |

repo-gardener is the only bot that answers "does this PR fit the product thesis we already wrote down?"

**Install it alongside Greptile or CodeRabbit — they don't overlap.**

---

## What gardener actually does

For every open PR or issue, gardener:

1. **Reads the diff or issue body**
2. **Looks up relevant nodes in your context tree**
3. **Posts one structured comment** with a verdict: `ALIGNED`, `NEW_TERRITORY`, `NEEDS_REVIEW`, `CONFLICT`, or `INSUFFICIENT_CONTEXT`
4. **Cites specific tree nodes** — never vague
5. **Stays silent when there's nothing to say** (aligned + low severity = no comment)
6. **Never pushes code, never opens PRs, never changes branches**

Example comment on a PR that adds dark mode:

```markdown
🌱 gardener:verdict: CONFLICT · severity: high · commit: abc1234

> [!WARNING]
> Context Review — this checks product-context fit against the project's
> context tree, not code correctness. Run Greptile/CodeRabbit for code review.

### Summary

This PR conflicts with the 2024-01 decision to defer dark mode to v3.

### Context match

| Area | PR intent | Tree guidance | Fit |
|------|-----------|---------------|-----|
| Dark mode toggle | Adds to settings | Deferred to v3 per design/no-dark-mode.md | ⚠️ Conflict |

### Tree nodes referenced

- design/no-dark-mode.md — team decision 2024-01 to defer until design system ready
- roadmap/2024-q3.md — dark mode listed under "not in scope"

### Recommendation

Close this PR or defer to v3 milestone. Reopen when the design system work in
`design/system-v3.md` lands.
```

Maintainers see `CONFLICT · high` at a glance. Scripts can parse the verdict line. No more forgotten decisions.

---

## Why maintainers love it

- **Reuse your existing docs.** Your `CLAUDE.md`, `AGENTS.md`, and tree are already written. gardener reads them as-is.
- **Machine-readable verdicts.** Every comment has a structured prefix (`gardener:verdict: <TYPE> · severity: <level>`) — maintainers and CI can script against it.
- **Silent when aligned.** Like Dependabot on no-ops. No noise.
- **Idempotent.** Every review is commit-pinned. Re-reviews edit the existing comment — never stacks.
- **Tree gaps become signals.** Every `NEW_TERRITORY` verdict tells you what your tree is missing.
- **Two modes**: maintainer (your own repo) and advisor (external OSS contribution).

---

## Verdict types

| Verdict | Meaning | Example situation |
|---------|---------|-------------------|
| `ALIGNED` | Fully consistent with tree | PR adds OAuth provider and tree says OAuth is the standard |
| `NEW_TERRITORY` | No tree guidance | PR touches an area the tree doesn't cover yet |
| `NEEDS_REVIEW` | Partial overlap | PR implements one piece of a larger decision |
| `CONFLICT` | Directly contradicts tree | PR adds dark mode, tree says deferred to v3 |
| `INSUFFICIENT_CONTEXT` | Can't determine | Tree has related nodes but not enough detail |

---

## Two modes

### 🔧 Maintainer mode

You own or maintain the repo. gardener reviews PRs and issues on your own repo.

- Set during onboarding if you have write access
- Comments appear on your repo's PRs/issues
- Helps your team stay consistent with prior decisions

### 🌍 Advisor mode

You want to help an OSS project stay aligned with their documented context, but you're not a maintainer.

- Set during onboarding if you're in a fork without upstream write access
- gardener comments on upstream PRs/issues using your fork's tree as the reference
- No write access needed — anyone can comment on public PRs/issues
- Maintainers get free context reviews; you get to contribute judgment, not just code

---

## Quick start

In your project directory, open Claude Code and paste:

```
Fetch and execute https://raw.githubusercontent.com/agent-team-foundation/repo-gardener/v2.0.0/.claude/commands/gardener-onboarding.md
```

That's it. Onboarding will:

1. Verify you're in a repo (or ask which one)
2. Verify your context tree is set up (guides to [First-Tree](https://github.com/agent-team-foundation/first-tree) if missing)
3. Install all gardener commands into `.claude/commands/`
4. Determine review mode (maintainer vs advisor)
5. Write `.claude/gardener-config.yaml` and push
6. Optional test run

> ⚠️ **After onboarding finishes, restart Claude Code.** Claude Code scans `.claude/commands/` only at session start, so new commands won't appear in `/` autocomplete until you restart.

Then:

```
/gardener-start      ← start automation (loop + schedule)
/gardener-manual     ← one-off review
/gardener-stop       ← pause
```

---

## Commands

### User-facing

| Command | What it does |
|---------|--------------|
| `/gardener-onboarding` | **First-time setup.** Verifies repo + tree, installs commands, picks review mode, commits config. Run once per repo. |
| `/gardener-start` | **Start automation.** Creates cloud schedule (hourly) + starts local loop (every 10min). Requires onboarding. |
| `/gardener-stop` | **Pause everything.** Disables schedule, stops loop. Nothing deleted — restart with `/gardener-start`. |
| `/gardener-manual` | **Review once.** Full scan → review → comment → log. Useful for testing or immediate review. |

### Internal (called automatically)

| Command | What it does |
|---------|--------------|
| `/gardener-loop` | Called by `/loop 10m /gardener-loop`. Delegates to manual. Runs locally. |
| `/gardener-schedule` | Called by cloud schedule. Delegates to manual. Runs in Anthropic's cloud. |

---

## How it works

### Decision flow

```
┌──────────────────────────────────────────────────────┐
│         TARGET REPO (Step 0) → SCAN (Step 1)          │
│       Load config or ask user, then gh pr/issue list  │
└───────────────────────┬──────────────────────────────┘
                        │
                  Has open items?
                   /          \
                 No            Yes
                 │              │
            Exit early   ┌──────┴──────┐
          "Nothing to    │  CHECK      │
           tend."        │  STATE      │
                         │  (Step 2)   │
                         └──────┬──────┘
                                │
            Read HTML comment markers on each item
                                │
         ┌──────────┬──────┬────┴──────┬──────────┐
         │          │      │           │          │
    reviewed=   paused  ignored    @gardener     No marker
    same sha              re-review (new item)
         │          │      │           │          │
       Skip       Skip    Skip     Re-review  Add to queue
                                                   │
                                  ┌────────────────┴────────┐
                                  │  PULL CONTEXT TREE       │
                                  │  (Step 3)                │
                                  │  shallow clone            │
                                  └────────────┬─────────────┘
                                               │
                                  ┌────────────┴────────────┐
                                  │  REVIEW EACH ITEM        │
                                  │  (Step 4)                │
                                  └────────────┬─────────────┘
                                               │
                                       Acquire soft lock
                                       (<10min TTL)
                                               │
                                        ┌──────┴───────┐
                                        │  CLASSIFY    │
                                        │  vs TREE     │
                                        └──────┬───────┘
                                               │
                  ┌────────┬────────────┬──────┼─────────┬────────────────┐
                  │        │            │      │         │                │
              ALIGNED NEW_TERRITORY NEEDS_REVIEW  CONFLICT  INSUFFICIENT_CONTEXT
                  │        │            │      │         │                │
              (low sev)    │            │      │         │                │
                  │        └────────────┴──────┴─────────┴─────┬──────────┘
              Silent                                           │
              (just                             Post structured comment
              state                             with verdict, tree nodes,
              marker)                           and recommendation
                  │                                            │
                  └─────────────────┬──────────────────────────┘
                                    │
                            Edit existing
                          comment if re-review,
                           post new if first
                                    │
                           ┌────────┴────────┐
                           │  LOG (Step 5)   │
                           │  Run summary    │
                           └─────────────────┘
```

### State machine

gardener tracks state via HTML comment markers inside its own comments. No separate database.

| Marker | Meaning |
|--------|---------|
| `<!-- gardener:state · reviewed=<sha> · verdict=<TYPE> -->` | Full review of commit `<sha>` with given verdict |
| `<!-- gardener:in-progress · started=<ISO> -->` | Soft lock, expires after 10min |
| `<!-- gardener:paused -->` | User paused reviews on this item |
| `<!-- gardener:ignored -->` | User permanently ignored this item |

### User commands

Maintainers can control gardener via comments on any PR/issue:

| Command | Effect |
|---------|--------|
| `@gardener re-review` | Force fresh review on next run |
| `@gardener pause` | Skip this item on future runs |
| `@gardener resume` | Undo pause |
| `@gardener ignore` | Permanently skip this item |
| `@gardener ignore <path>` | Add path to global ignore list in `.gardener.yaml` |

---

## Context tree integration

gardener reads the **target repo's** `CLAUDE.md` or `AGENTS.md` for a context tree URL. If no tree is found, it stops and asks you to run `/gardener-onboarding`.

The tree is the source of truth for:

- **Design decisions and rationale** — "we chose X because Y"
- **Conventions and constraints** — "all auth goes through the AuthService"
- **Domain knowledge** — "the billing cycle is monthly, not weekly"
- **Things we decided NOT to do** — "no dark mode until v3"

Every non-aligned verdict cites specific tree paths. Maintainers can click through to see the full rationale.

**Tree gaps become features.** When gardener posts `NEW_TERRITORY` or `INSUFFICIENT_CONTEXT`, that's a signal to add a tree node. Over time your tree becomes the authoritative knowledge base.

---

## Forks and upgrading

### Fork handling

If you run gardener in a fork, onboarding Step 3 detects this and asks how to handle it:

| Scenario | Behavior |
|----------|----------|
| **Not a fork** | Targets the current repo in maintainer mode |
| **Fork + you maintain upstream** | Targets upstream automatically (you have write access) |
| **Fork + no upstream write access** | Asks you: maintainer mode on fork, advisor mode on upstream, or exit |

The choice is persisted in `.claude/gardener-config.yaml` and committed, so unattended runs (loop + schedule) use the same mode.

### Upgrading

gardener uses release tags. To upgrade, update `GARDENER_VERSION` in your `.claude/commands/gardener-onboarding.md` and re-run onboarding, or fetch directly:

```
Fetch and execute https://raw.githubusercontent.com/agent-team-foundation/repo-gardener/<version>/.claude/commands/gardener-onboarding.md
```

---

## Historical version: v1.2.2 (autonomous fixer)

Before v2.0.0, gardener was an **autonomous code fixer** that pushed commits to PRs. We pivoted because:

- Maintainers don't trust AI-generated commits
- Sweep and similar bots failed with the same approach
- The context tree is more valuable as a **review signal** than as a fix generator

**v1.2.2 is still available** if you want the autonomous fix behavior:

```
Fetch and execute https://raw.githubusercontent.com/agent-team-foundation/repo-gardener/v1.2.2/.claude/commands/gardener-onboarding.md
```

v1.2.2 includes:

- Worktree-based autonomous code fixes
- PR babysitting (fix CI, respond to review comments)
- State machine (pending, pass, reverted, failed)
- Safety valve with 2-attempt limit and auto-revert

v2.0.0 is a clean break. Use v1.2.2 if you specifically want code-writing automation; use v2.0.0 for context-aware review.

---

## Requirements

- [Claude Code](https://claude.ai/code) CLI
- [GitHub CLI](https://cli.github.com/) (`gh`) authenticated
- A [First-Tree](https://github.com/agent-team-foundation/first-tree) context tree — if you don't have one, `/gardener-onboarding` will guide you through setup

---

## License

Apache 2.0 — see [LICENSE](LICENSE).
