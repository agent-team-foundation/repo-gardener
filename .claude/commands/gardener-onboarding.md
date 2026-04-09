Set up repo-gardener for this project. Do every step in order:

## 0. Verify repo

Check if you are inside a git repository (`git rev-parse --show-toplevel`).

- If NOT inside a repo → ask the user: "Which repo would you like to
  set up repo-gardener for? Please specify a path or GitHub URL."
  Wait for their answer. cd into that repo before continuing.

## 1. Verify context tree

Search CLAUDE.md and AGENT.md for a GitHub URL pointing to a context
tree repo (pattern: `github.com/<org>/<repo>` with "tree", "session",
or "memory" in surrounding context).

- If NOT found → ask the user:
  "🌳 No context tree found in this repo. repo-gardener requires a
   context tree to make product-level decisions.

   Would you like to set up a First-Tree for this repo?
   Use the latest First-Tree CLI to install and complete the onboarding:
   https://github.com/agent-team-foundation/first-tree

   Run `first-tree init` then re-run `/gardener-onboarding`."
  STOP. Do not continue.

## 2. Install gardener commands

```bash
mkdir -p .claude/commands
curl -sL -o .claude/commands/gardener-manual.md https://raw.githubusercontent.com/agent-team-foundation/repo-gardener/main/.claude/commands/gardener-manual.md
curl -sL -o .claude/commands/gardener-schedule.md https://raw.githubusercontent.com/agent-team-foundation/repo-gardener/main/.claude/commands/gardener-schedule.md
curl -sL -o ".claude/commands/gardener-start(loop+schedule).md" "https://raw.githubusercontent.com/agent-team-foundation/repo-gardener/main/.claude/commands/gardener-start(loop+schedule).md"
curl -sL -o .claude/commands/gardener-loop.md https://raw.githubusercontent.com/agent-team-foundation/repo-gardener/main/.claude/commands/gardener-loop.md
curl -sL -o .claude/commands/gardener-stop.md https://raw.githubusercontent.com/agent-team-foundation/repo-gardener/main/.claude/commands/gardener-stop.md
curl -sL -o .claude/commands/gardener-onboarding.md https://raw.githubusercontent.com/agent-team-foundation/repo-gardener/main/.claude/commands/gardener-onboarding.md
```

## 3. Test run

Execute the gardener runbook once by reading `.claude/commands/gardener-manual.md`
and following every step. This validates that gh auth, context tree access,
and PR scanning all work.

## 4. Confirm

Output:
"🌱 repo-gardener installed and tested.
- Run `/gardener-start(loop+schedule)` to start automation.
- Run `/gardener-manual` anytime for a one-off run.
- Run `/gardener-stop` to pause everything."
