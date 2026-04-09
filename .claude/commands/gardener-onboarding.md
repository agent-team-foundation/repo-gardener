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

Fetch from a pinned commit for integrity. Check
https://github.com/agent-team-foundation/repo-gardener/releases
for the latest pinned version.

```bash
GARDENER_SHA="51e3bcd66f53192ca98ab25398eff00f1102eaf3"
BASE="https://raw.githubusercontent.com/agent-team-foundation/repo-gardener/${GARDENER_SHA}"
mkdir -p .claude/commands
curl -sL -o .claude/commands/gardener-manual.md "${BASE}/.claude/commands/gardener-manual.md"
curl -sL -o .claude/commands/gardener-schedule.md "${BASE}/.claude/commands/gardener-schedule.md"
curl -sL -o ".claude/commands/gardener-start(loop+schedule).md" "${BASE}/.claude/commands/gardener-start(loop+schedule).md"
curl -sL -o .claude/commands/gardener-loop.md "${BASE}/.claude/commands/gardener-loop.md"
curl -sL -o .claude/commands/gardener-stop.md "${BASE}/.claude/commands/gardener-stop.md"
curl -sL -o .claude/commands/gardener-onboarding.md "${BASE}/.claude/commands/gardener-onboarding.md"
```

## 3. Commit and push

The command files must be in the remote repo so that `/schedule`
(which runs in the cloud) can access them.

```bash
git add .claude/commands/gardener-*.md
git diff --cached --quiet && echo "No changes to commit" || git commit -m "chore: install repo-gardener commands"
git push
```

If push fails (e.g. branch protection), tell the user:
"Push failed — your branch may have protection rules. Create a PR
with these changes instead, or push to a different branch."

## 4. Test run

Ask the user:
"⚠️ The test run will process **real** open PRs and issues — it is not
a dry run. If you have open items, the agent will attempt to fix them.

Proceed with the live test run?"

- If the user says yes → execute the gardener runbook once by reading
  `.claude/commands/gardener-manual.md` and following every step.
  This validates that gh auth, context tree access, and PR scanning all work.
- If the user says no → skip the test run and proceed to Step 5.

## 5. Confirm

Output:
"🌱 repo-gardener installed and tested.
- Commands committed and pushed to remote.
- Run `/gardener-start(loop+schedule)` to start automation.
- Run `/gardener-manual` anytime for a one-off run.
- Run `/gardener-stop` to pause everything."
