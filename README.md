<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/logo-black-bg.png">
    <source media="(prefers-color-scheme: light)" srcset="assets/logo-white-bg.png">
    <img src="assets/logo-white-bg.png" alt="repo-gardener" width="128">
  </picture>
</p>

# repo-gardener

**Context-aware review bot + tree sync for GitHub.** Reviews PRs against your context tree and keeps the tree up to date.

> 🎯 **Two modules, one bot.**
> **Comment** — reviews source repo PRs for context fit. **Sync** — detects source drift and opens tree-update PRs.

[![Verdict accuracy](https://img.shields.io/badge/verdict%20accuracy-84.6%25-brightgreen)](https://github.com/agent-team-foundation/gardener-bench)

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
| **🌱 repo-gardener** | **Product context fit + tree sync** | **Verdict comments + tree-update PRs** | **Yes — reads and updates the context tree** |

repo-gardener is the only bot that answers "does this PR fit the product thesis we already wrote down?"

**Install it alongside Greptile or CodeRabbit — they don't overlap.** gardener reviews every item regardless of whether other bots have already commented. A PR can have:
- A Greptile comment saying "code is clean"
- A CodeRabbit walkthrough
- A gardener verdict saying "conflicts with `product/no-dark-mode.md`"

Each bot is reviewing a different dimension. We post on items with linked PRs too — a fix PR can still deserve a context review.

---

## Accuracy

We grade every verdict gardener posts against what the maintainer actually did with the PR (merged / merged-after-revision / rejected). Scoring and dashboards live in [gardener-bench](https://github.com/agent-team-foundation/gardener-bench).

**Latest run — 2026-04-17, [paperclipai/paperclip](https://github.com/paperclipai/paperclip):**

| Verdict accuracy | Correct | Partial | Wrong | Scorable PRs |
|---|---|---|---|---|
| **84.6%** | 11 | 0 | 2 | 13 of 231 observed |

Full breakdown: [accuracy.json](https://github.com/agent-team-foundation/gardener-bench/blob/main/reports/paperclipai-paperclip/2026-04-17/accuracy.json) · live dashboard: https://gardener-report.pages.dev

---

## What gardener actually does

### Comment module

For every open PR or issue on source repos, gardener:

1. **Reads the diff or issue body**
2. **Looks up relevant nodes in your context tree**
3. **Posts one structured comment** with a verdict: `ALIGNED`, `NEW_TERRITORY`, `NEEDS_REVIEW`, `CONFLICT`, or `INSUFFICIENT_CONTEXT`
4. **Cites specific tree nodes** — never vague
5. **Stays silent when there's nothing to say** (aligned + low severity = no comment)

### Sync module

When source repo PRs get merged, gardener:

1. **Detects drift** between the tree and source repos (via `.first-tree/bindings/`)
2. **Classifies each merged PR** as `TREE_OK` (already covered), `TREE_MISS` (new area), or `TREE_SUPPLEMENT` (existing node needs additions)
3. **Opens one tree PR per source PR** with new NODE.md files for TREE_MISS items
4. **Reviews its own sync PRs** for frontmatter format, ownership consistency, member dedup, content alignment, and sibling/parent consistency
5. **PR author becomes node owner** — auto-creates member if not in tree

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

<sub>Reviewed commit: <code>abc1234</code> · Tree snapshot: <code>def5678</code> · Commands: <code>@gardener re-review</code></sub>

<sub>🌱 Posted by [repo-gardener](https://github.com/agent-team-foundation/repo-gardener) — an open-source context-aware review bot built on [First-Tree](https://github.com/agent-team-foundation/first-tree). Reviews this repo against [acme/tree](https://github.com/acme/tree), a user-maintained context tree. Not affiliated with this project's maintainers.</sub>
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

## Installation

Gardener installs into your **tree repo** (the Context Tree). It auto-detects source repos from `.first-tree/bindings/`.

The config file (`.claude/gardener-config.yaml`) is auto-generated:

```yaml
tree_repo: serenakeyitan/paperclip-tree
installed_version: v2.3.2
target_repos:
  - paperclipai/paperclip
  - paperclipai/docs
```

`target_repos` are extracted from the `remoteUrl` field in each `.first-tree/bindings/*.json` file. You don't need to configure them manually.

---

## Quick start

In your **tree repo** directory, open Claude Code and paste:

```
Fetch the latest release of repo-gardener and execute its onboarding script: https://github.com/agent-team-foundation/repo-gardener/releases/latest
```

That's it. Onboarding will:

1. Verify you're in a Context Tree repo (`.first-tree/` exists)
2. Read source repos from `.first-tree/bindings/` (auto-detected)
3. Check prerequisites (`gh` CLI, `claude` CLI)
4. Install all 16 gardener commands into `.claude/commands/`
5. Auto-generate `.claude/gardener-config.yaml` from bindings
6. Commit and push

> ⚠️ **After onboarding finishes, restart Claude Code.** Claude Code scans `.claude/commands/` only at session start, so new commands won't appear in `/` autocomplete until you restart.

Then:

```
/gardener-start          ← start all modules (comment + sync)
/gardener-stop           ← stop all modules
/gardener-comment-manual ← one-off PR review
/gardener-sync-manual    ← one-off tree sync
/gardener-comment-watch  ← live terminal log popup with clickable URLs
/gardener-upgrade        ← auto-update to latest release
```

---

## Commands

### User-facing

| Command | What it does |
|---------|-------------|
| `/gardener-onboarding` | First-time setup |
| `/gardener-start` | Start all modules (comment + sync) |
| `/gardener-stop` | Stop all modules |
| `/gardener-comment-start` | Start comment module only |
| `/gardener-comment-stop` | Stop comment module only |
| `/gardener-comment-manual` | One-off PR review |
| `/gardener-comment-watch` | Live log popup |
| `/gardener-sync-start` | Start sync module only |
| `/gardener-sync-stop` | Stop sync module only |
| `/gardener-sync-manual` | One-off tree sync |
| `/gardener-sync-watch` | Live sync progress log |
| `/gardener-upgrade` | Auto-update to latest release |

### Internal (called automatically)

| Command | What it does |
|---------|-------------|
| `/gardener-comment-loop` | Called by `/loop`. Delegates to comment-manual. |
| `/gardener-comment-schedule` | Called by cloud schedule. Delegates to comment-manual. |
| `/gardener-sync-loop` | Called by `/loop`. Delegates to sync-manual. |
| `/gardener-sync-schedule` | Called by cloud schedule. Delegates to sync-manual. |

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
        Filter: only paths_ignored (no linked-PR filter)
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

### Sync flow

```
Step 0: READ BINDINGS
        .first-tree/bindings/*.json → list of source repos
              │
              ▼
Step 1: DETECT DRIFT (per source)
        Compare lastReconciledSourceCommit to HEAD
        Fetch merged PRs since last sync
              │
       Has new merged PRs?
         /        \
       No          Yes
        │           │
      Skip          ▼
              Step 2: CLASSIFY (per merged PR, concurrency 10)
                     Claude CLI classifies each PR:
                       ├─ TREE_OK → skip
                       ├─ TREE_MISS → new NODE.md
                       └─ TREE_SUPPLEMENT → suggestion in PR body
                     │
                     ▼
              Step 3: APPLY (per PR with proposals)
                     Create branch, write NODE.md, push, open PR
                     Label: first-tree:sync
                     Footer: @gardener re-sync
                     │
                     ▼
              Step 4: REVIEW SYNC PRs
                     Check frontmatter, ownership, member dedup,
                     content alignment, sibling consistency
                     Post review comment on each PR
                     │
                     ▼
              Step 5: HOUSEKEEPING PR
                     Pin lastReconciledSourceCommit
                     Regenerate CODEOWNERS
                     (merge after all sync PRs land)
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

| Command | Effect |
|---------|--------|
| `@gardener re-review` | Force fresh comment review on next run |
| `@gardener re-sync` | Close sync PR and regenerate on next run |

---

## Context tree integration

Gardener reads the tree from the repo where it's installed. Source repos are auto-detected from `.first-tree/bindings/`. The tree is the source of truth for:

- **Design decisions and rationale** — "we chose X because Y"
- **Conventions and constraints** — "all auth goes through the AuthService"
- **Domain knowledge** — "the billing cycle is monthly, not weekly"
- **Things we decided NOT to do** — "no dark mode until v3"

Every non-aligned verdict cites specific tree paths. Maintainers can click through to see the full rationale.

**Tree gaps become features.** When gardener posts `NEW_TERRITORY` or `INSUFFICIENT_CONTEXT`, that's a signal to add a tree node. Over time your tree becomes the authoritative knowledge base.

---

## Upgrading

Run `/gardener-upgrade` from any project that has gardener installed. It will:

1. Read your `installed_version` from `.claude/gardener-config.yaml`
2. Resolve the latest release tag via the GitHub API
3. Fetch updated command files
4. Update `installed_version` in the config
5. Commit and push to your config repo

Then restart Claude Code so the new commands are picked up.

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
- [Claude CLI](https://docs.anthropic.com/en/docs/claude-code) (`claude`) — used by sync for AI classification
- [GitHub CLI](https://cli.github.com/) (`gh`) authenticated
- A [First-Tree](https://github.com/agent-team-foundation/first-tree) context tree with at least one bound source repo

---

## License

Apache 2.0 — see [LICENSE](LICENSE).
