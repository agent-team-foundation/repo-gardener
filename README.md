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

Example comment on a PR that adds dark mode (this is the exact format gardener produces):

```markdown
<!-- gardener:state · reviewed=abc1234 · verdict=CONFLICT · severity=high · tree_sha=def5678 -->

**🌱 gardener:verdict:** `CONFLICT` · severity: `high` · commit: `abc1234`

> [!WARNING]
> **Context Review** — this checks product-context fit against the project's
> context tree, not code correctness. Run Greptile/CodeRabbit for code review.

### Summary

This PR conflicts with the 2024-01 decision to defer dark mode to v3.

<details open>
<summary><strong>Context match</strong></summary>

| Area | Item intent | Tree guidance | Fit |
|------|-------------|---------------|-----|
| Dark mode toggle | Adds to settings page | Deferred to v3 per design/no-dark-mode.md | ⚠️ Conflict |

</details>

<details>
<summary><strong>Tree nodes referenced</strong></summary>

- [`design/no-dark-mode.md`](https://github.com/acme/tree/blob/main/design/no-dark-mode.md) — team decision 2024-01 to defer until design system ready
- [`roadmap/2024-q3.md`](https://github.com/acme/tree/blob/main/roadmap/2024-q3.md) — dark mode listed under "not in scope"

</details>

### Recommendation

Close this PR or defer to v3 milestone. Reopen when the design system work in `design/system-v3.md` lands.

---

<sub>Reviewed commit: <code>abc1234</code> · Tree snapshot: <code>def5678</code> · Commands: <code>@gardener re-review</code> · <code>@gardener pause</code> · <code>@gardener ignore</code></sub>
```

**The first line is an HTML comment** — invisible to humans, but machine-readable so CI/scripts can grep for `gardener:state` or `verdict=CONFLICT`. The second line is the human-visible verdict.

Maintainers see `CONFLICT · high` at a glance. Scripts can parse the verdict line. No more forgotten decisions.

---

## Why maintainers love it

- **Reuse your existing docs.** Your `CLAUDE.md`, `AGENTS.md`, and tree are already written. gardener reads them as-is.
- **Machine-readable verdicts.** Every comment has a structured prefix (`gardener:verdict: <TYPE> · severity: <level>`) — maintainers and CI can script against it.
- **Silent when aligned.** Like Dependabot on no-ops. No noise.
- **Idempotent.** Every review is commit-pinned. Re-reviews edit the existing comment — never stacks.
- **Tree gaps become signals.** Every `NEW_TERRITORY` verdict tells you what your tree is missing.
- **Works on any public repo** — your own or upstream OSS. Anyone can comment on public PRs/issues, so gardener doesn't need write access to contribute context reviews.

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

## Two use cases, one mode

gardener has a single review mode. You pick which repo to target during onboarding. Because gardener only posts comments (which anyone can do on public PRs/issues), the same mode works for both cases:

### Reviewing your own repo

You're a maintainer. During onboarding, target your own repo. gardener comments on your team's PRs and issues, helping you stay consistent with prior decisions.

- Full write access lets gardener use the `gardener:reviewed` label for the silent-aligned path (no comment needed when everything is fine)
- Clean conversations — silent when aligned, structured when not

### Reviewing an external OSS repo

You want to contribute context reviews to an upstream OSS project. During onboarding, target that repo instead. You don't need to fork it — gardener only reads + comments.

- No write access required — comments work on any public repo you can read
- The silent-aligned label falls back to a minimal aligned comment (since you can't edit labels on external repos)
- Maintainers get free context reviews; you contribute judgment instead of code

---

## Quick start

In your project directory, open Claude Code and paste:

```
Fetch and execute https://raw.githubusercontent.com/agent-team-foundation/repo-gardener/v2.0.0/.claude/commands/gardener-onboarding.md
```

That's it. Onboarding will:

1. Verify you're in a git repo
2. Verify your context tree is set up (guides to [First-Tree](https://github.com/agent-team-foundation/first-tree) if missing)
3. Install all gardener commands into `.claude/commands/`
4. Ask which repo to target (this repo or a different one)
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
| `/gardener-onboarding` | **First-time setup.** Verifies repo + tree, installs commands, asks which repo to target, commits config. Run once per repo. |
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
Step 0: LOAD OR DETERMINE TARGET REPO
        Read .claude/gardener-config.yaml
        (or ask user which repo to target)
              │
              ▼
Step 1: SCAN FOR WORK
        gh pr list + gh issue list (limit 30)
        Filter: issues with linked PRs, paths_ignored
              │
       Has open items?
         /        \
       No          Yes
        │           │
      Exit          ▼
              Step 2: CHECK STATE (per item)
                     Fetch all comments via gh api
                     Single JSON pass → classify:
                       ├─ gardener:ignored → skip forever
                       ├─ gardener:paused (no resume) → skip run
                       ├─ @gardener re-review (newer than state) → force
                       ├─ gardener:state, sha matches HEAD → skip
                       ├─ gardener:state, sha differs → re-review
                       │   (remember comment ID for PATCH)
                       └─ no marker → first review
                     │
                     ▼
Step 3: PULL CONTEXT TREE (only if queue non-empty)
        grep CLAUDE.md/AGENTS.md for GitHub URL
        shallow clone into .gardener-tree-cache/
        Capture tree commit SHA
                     │
                     ▼
Step 4: REVIEW EACH ITEM
        ┌────────────────────────────┐
        │ 4a: Acquire lock (eyes      │
        │     reaction, 10min TTL)    │
        │ 4b: Classify vs tree →      │
        │     5 verdict types         │
        │ 4c: Silent path:            │
        │     ALIGNED + low severity  │
        │     → apply gardener:       │
        │     reviewed label, no      │
        │     comment                 │
        │ 4d: Build comment with      │
        │     HTML marker + visible   │
        │     verdict line + table    │
        │ 4e: POST new OR PATCH       │
        │     existing via gh api     │
        │     -X PATCH /issues/       │
        │     comments/<id>           │
        │ 4f: Handle @gardener        │
        │     commands + remove       │
        │     eyes reaction           │
        └────────────────────────────┘
                     │
                     ▼
Step 5: LOG TO STDOUT
        Run summary (counts per verdict,
        skipped, tree SHA) — no comment spam
```

### State tracking

gardener uses three mechanisms, no database:

| Mechanism | Used for |
|-----------|----------|
| **HTML comment markers** inside gardener's own comment | Review state (`<!-- gardener:state · reviewed=<sha> · verdict=<TYPE> · severity=<lvl> · tree_sha=<sha> -->`), pause, ignore |
| **`eyes` reaction** on the PR/issue | Soft lock while processing (10min TTL) |
| **`gardener:reviewed` label** on the PR/issue | Silent-aligned path (ALIGNED + low severity — no comment posted) |

Editing in place:
- On re-review, gardener finds its prior comment via `gh api /repos/{owner}/{repo}/issues/{num}/comments` and PATCHes it using `gh api -X PATCH /repos/{owner}/{repo}/issues/comments/{id}`.
- Never posts duplicates.

### User commands

Maintainers can control gardener via comments on any PR/issue:

| Command | Effect |
|---------|--------|
| `@gardener re-review` | Force fresh review on next run |
| `@gardener pause` | Skip this item on future runs |
| `@gardener resume` | Undo pause |
| `@gardener ignore` | Permanently skip this item |
| `@gardener ignore <path>` | Add path to `paths_ignored:` in `.claude/gardener-config.yaml` |

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

## Upgrading

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
