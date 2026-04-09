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
- A [First-Tree](https://github.com/agent-team-foundation/first-tree) context tree — if you don't have one, `/gardener-onboarding` will guide you through setup

## Usage

### As a slash command

Copy `.claude/commands/gardener-manual.md` into your repo's `.claude/commands/` directory, then:

```
/gardener-manual
```

### Recommended: run both loop + schedule together

```bash
# One-time setup — cloud agent as always-on fallback
/schedule every hour /gardener-schedule

# When you're at your computer — local agent for faster cycles
/loop 10m /gardener-loop
```

**You don't need to switch between them.** They coexist safely:

| | `/loop` | `/schedule` |
|---|---|---|
| Runs where | Your machine | Anthropic cloud |
| Frequency | Every 10min (configurable) | Every hour |
| When you close laptop | Stops | Keeps running |
| Auth | Your local `gh auth` | GitHub MCP connector |

The concurrency lock (`gardener:in-progress` comment) prevents both from
touching the same PR at the same time. If loop just handled a PR, schedule
sees the lock and skips it. If you close your laptop, schedule picks up
on the next hourly run.

### Just schedule (cloud only)

```
/schedule every hour /gardener-schedule
```

Runs in Anthropic's cloud — works even when your machine is off.

### Just loop (local only)

```
/loop 10m /gardener-loop
```

Faster cycles, uses your local `gh auth`. Stops when you close your laptop.

## How it works

### Decision flow

```
┌─────────────────────────────────────────────────────────┐
│                    SCAN (Step 0)                         │
│         gh pr list + gh issue list                       │
└──────────────────────┬──────────────────────────────────┘
                       │
                 Has open items?
                  /          \
                No            Yes
                │              │
           Exit early    ┌─────┴─────┐
          "Nothing       │ CHECK     │
           to tend."     │ STATE     │
                         │ (Step 1)  │
                         └─────┬─────┘
                               │
              Read gardener comments on each item
                               │
         ┌─────────┬───────┬───┴────┬──────────┐
         │         │       │        │          │
    :in-progress :pending :pass :reverted :failed
    (< 30min)      │       │       │          │
         │         │       │       │          │
       Skip    Check CI  Skip    Skip      Skip
               result      │       │          │
              /      \     └───────┴──────────┘
           Pass    Fail          (needs human)
            │     /     \
       Mark  Same err  New err
       :pass    │        │
             Revert   Re-queue
             mark      for fix
           :reverted
                               │
                         ┌─────┴─────┐
                         │ PRIORITIZE │
                         │ (Step 2)   │
                         └─────┬─────┘
                               │
                    1. Prior pending (new err)
                    2. CI failing PRs
                    3. PRs w/ review comments
                    4. Bug issues (w/ repro)
                    5. Feature issues
                               │
                   ┌───────────┴───────────┐
                   │ PULL CONTEXT TREE      │
                   │ (Step 3 — only if      │
                   │  queue has context-     │
                   │  informed items)        │
                   └───────────┬────────────┘
                               │
                         Found tree?
                        /          \
                      No            Yes
                      │              │
                 Direct fixes   Clone tree
                 only (skip      (--depth 1)
                 context items)      │
                      │              │
                      └──────┬───────┘
                             │
                    ┌────────┴────────┐
                    │ PROCESS (Step 4) │
                    │ For each item:   │
                    └────────┬────────┘
                             │
                     Acquire lock
                  (gardener:in-progress)
                             │
                        ┌────┴────┐
                        │ TRIAGE  │
                        └────┬────┘
                             │
                    ┌────────┴────────┐
                    │                 │
              Direct fix      Context-informed
              (no tree)         (read tree)
                    │                 │
             Spawn worktree    Spawn worktree
               agent             agent
                    │                 │
              Fix + commit      Tree has guidance?
              + push            /              \
                    │         Yes               No
                    │          │                 │
                    │    Fix + commit     Comment: hand
                    │    + push +         to human +
                    │    cite node        suggest node
                    │          │                 │
                    │     :pending          :failed
                    │          │                 │
               :pending        └────────┬───────┘
                    │                   │
                    └───────────────────┘
                             │
                    Move to next item
                   (DO NOT wait for CI)
                             │
                    ┌────────┴────────┐
                    │  LOG (Step 5)    │
                    │  Summary comment │
                    │  + tree gaps     │
                    └─────────────────┘
```

### Issue-specific flow

```
Issue comes in
       │
  Has label "bug" + repro steps?
  /          |              \
Yes      "feature"      No label/vague
 │           │               │
Create    Read tree:      Comment asking
branch    in scope?       for clarification
+ fix        │               │
+ PR      /     \          Skip
        Yes      No
         │        │
       Create   Skip +
       branch   comment
       + fix
       + PR
```

### Safety valve (revert flow)

```
gardener:pending item from prior run
         │
    CI passing?
    /         \
  Yes          No
   │            │
Mark         Is HEAD my commit?
:pass        /              \
           Yes               No
            │                 │
     Same error?         Don't revert
     /        \          Comment warning
   Yes         No
    │           │
  Revert    Re-queue
  mark      (attempt 2)
:reverted       │
            Fail again?
            /        \
          Yes         No
           │           │
     Mark :failed   Mark :pass
     (needs human)
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

repo-gardener reads a URL from your `CLAUDE.md` or `AGENT.md` pointing to your context tree repo. If no tree is found, it stops and asks you to run `/gardener-onboarding`.

For direct fixes (type errors, lint, build), no tree is needed. For anything that requires product judgment, the agent reads the tree and either:

- **Decides autonomously** if the tree has enough context (cites the tree node)
- **Hands back to human** if the tree lacks guidance, and suggests what node to add

Every handback is a signal. Fill the gap, and next time the agent handles it alone.

## Quick start

In your project directory, open Claude Code and paste:

```
Fetch and execute https://raw.githubusercontent.com/agent-team-foundation/repo-gardener/main/.claude/commands/gardener-onboarding.md
```

That's it. It will:
1. Verify you're in a repo (or ask which one)
2. Verify your context tree is set up
3. Install all gardener commands into `.claude/commands/`
4. Commit and push commands to remote (so `/schedule` can access them)
5. Run a test pass

Then start automation:

```
/gardener-start(loop+schedule)
```

After onboarding, you have these commands available:

```
/gardener-onboarding          ← first-time setup (install + test)
/gardener-start(loop+schedule) ← start loop + schedule
/gardener-stop                 ← pause everything
/gardener-manual               ← manual one-time run
```

## Commands & files

### User-facing commands

| Command | What it does |
|---------|-------------|
| `/gardener-onboarding` | **First-time setup.** Checks if you're in a repo, verifies context tree exists (guides you to [First-Tree](https://github.com/agent-team-foundation/first-tree) if not), installs all command files, runs a test pass. Run once per repo. |
| `/gardener-start(loop+schedule)` | **Start automation.** Launches cloud schedule (every hour) + local loop (every 10min). Requires onboarding to be done first. |
| `/gardener-stop` | **Pause everything.** Stops the local loop and disables the cloud schedule. Nothing is deleted — restart with `/gardener-start(loop+schedule)`. |
| `/gardener-manual` | **Run once, right now.** Executes the full scan → triage → fix → log runbook a single time. Use this to test or to handle something immediately. |

### Internal commands (called automatically)

| Command | What it does |
|---------|-------------|
| `/gardener-loop` | Called by `/loop 10m /gardener-loop`. Short prompt that reads `gardener-manual.md` and executes it. Runs locally on your machine every 10 minutes. Stops when you close your laptop. |
| `/gardener-schedule` | Called by `/schedule every hour /gardener-schedule`. Same short prompt, but runs in Anthropic's cloud every hour. Works even when your machine is off. |

### Files

| File | Purpose |
|------|---------|
| `.claude/commands/gardener-manual.md` | The runbook — full step-by-step agent logic (scan, triage, worktree fix, revert, log) |
| `.claude/commands/gardener-loop.md` | Short prompt for `/loop` — tells agent to read and execute the runbook |
| `.claude/commands/gardener-schedule.md` | Short prompt for `/schedule` — same as loop but runs in cloud |
| `.claude/commands/gardener-onboarding.md` | First-time setup — verify repo, verify tree, install files, test run |
| `.claude/commands/gardener-start(loop+schedule).md` | Start automation — launch schedule + loop (requires onboarding first) |
| `.claude/commands/gardener-stop.md` | Teardown script — stop loop, disable schedule |

## License

MIT
