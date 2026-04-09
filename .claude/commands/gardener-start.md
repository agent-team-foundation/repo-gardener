Set up repo-gardener for this project. Do every step in order:

## 1. Verify context tree

Search CLAUDE.md and AGENT.md for a GitHub URL pointing to a context
tree repo (pattern: `github.com/<org>/<repo>` with "tree", "session",
or "memory" in surrounding context).

- If NOT found → output:
  "❌ No context tree found. Run `first-tree init` first, then re-run `/gardener-start`."
  STOP. Do not continue.

## 2. Install gardener commands

```bash
mkdir -p .claude/commands
curl -sL -o .claude/commands/gardener-manual.md https://raw.githubusercontent.com/agent-team-foundation/repo-gardener/main/.claude/commands/gardener-manual.md
curl -sL -o .claude/commands/gardener-schedule.md https://raw.githubusercontent.com/agent-team-foundation/repo-gardener/main/.claude/commands/gardener-schedule.md
curl -sL -o .claude/commands/gardener-start.md https://raw.githubusercontent.com/agent-team-foundation/repo-gardener/main/.claude/commands/gardener-start.md
curl -sL -o .claude/commands/gardener-stop.md https://raw.githubusercontent.com/agent-team-foundation/repo-gardener/main/.claude/commands/gardener-stop.md
```

## 3. Test run

Execute the gardener runbook once by reading `.claude/commands/gardener-manual.md`
and following every step. This validates that gh auth, context tree access,
and PR scanning all work.

## 4. Start schedule + loop

- Set up cloud schedule: `/schedule every hour /gardener-schedule`
- Start local loop: `/loop 10m /gardener-manual`

## 5. Confirm

Output:
"🌱 repo-gardener is running.
- Cloud schedule: every hour (runs when your machine is off)
- Local loop: every 10min (runs while you're here)
- Stop anytime: `/gardener-stop`"
