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

Fetch from a release tag for integrity and reproducibility.

```bash
GARDENER_VERSION="v1.2.0"
BASE="https://raw.githubusercontent.com/agent-team-foundation/repo-gardener/${GARDENER_VERSION}"
mkdir -p .claude/commands
curl -sL -o .claude/commands/gardener-manual.md "${BASE}/.claude/commands/gardener-manual.md"
curl -sL -o .claude/commands/gardener-schedule.md "${BASE}/.claude/commands/gardener-schedule.md"
curl -sL -o ".claude/commands/gardener-start(loop+schedule).md" "${BASE}/.claude/commands/gardener-start(loop+schedule).md"
curl -sL -o .claude/commands/gardener-loop.md "${BASE}/.claude/commands/gardener-loop.md"
curl -sL -o .claude/commands/gardener-stop.md "${BASE}/.claude/commands/gardener-stop.md"
curl -sL -o .claude/commands/gardener-onboarding.md "${BASE}/.claude/commands/gardener-onboarding.md"
```

Verify downloads are not empty (guards against network failure or 404):

```bash
for f in .claude/commands/gardener-*.md; do
  if [ ! -s "$f" ]; then
    echo "❌ Download failed: $f is empty. Check your network and try again."
    exit 1
  fi
done
```

## 3. Determine target repo mode (fork handling)

repo-gardener needs to know which repo to scan and how to handle fixes.
This is especially critical for forks — running unattended on a fork
could silently target the upstream repo with hundreds of strangers' PRs.

Run: `gh repo view --json nameWithOwner,isFork,parent,viewerPermission`

Decision tree:

1. **Not a fork** → `target_repo=<current>`, `fix_mode=direct`. No prompt needed.

2. **Fork + user has WRITE/MAINTAIN/ADMIN on parent** → user maintains
   upstream. `target_repo=<parent>`, `fix_mode=direct`. No prompt needed.

3. **Fork + no upstream write access** → STOP and ask the user:

   "🔀 This is a fork of `<parent>`. How should repo-gardener handle it?

   1. **Target my fork only** — scan issues/PRs on `<fork>`
      (useful if you use your fork for personal tracking)
   2. **Contribute to upstream** — scan `<parent>`, fix in fork, open cross-repo PRs
      (OSS contributor bot workflow)
   3. **Exit** — let me run gardener in a different repo

   Which would you like?"

   Wait for the answer.
   - Option 1 → `target_repo=<fork>`, `fix_mode=direct`
   - Option 2 → `target_repo=<parent>`, `fix_mode=fork-contribute`, `fork_owner=<fork-owner>`
   - Option 3 → exit cleanly

Write the chosen values to `.claude/gardener-config.yaml`:

```yaml
target_repo: <owner>/<name>
fix_mode: direct            # or fork-contribute
fork_owner: <owner>         # only if fix_mode=fork-contribute
```

## 4. Commit and push

The command files and config must be in the remote repo so that
`/schedule` (which runs in the cloud) can access them.

```bash
git add .claude/commands/gardener-*.md .claude/gardener-config.yaml
git diff --cached --quiet && echo "No changes to commit" || git commit -m "chore: install repo-gardener commands and config"
git push
```

If push fails (e.g. branch protection), tell the user:
"Push failed — your branch may have protection rules. Create a PR
with these changes instead, or push to a different branch."

## 6. Test run

Ask the user:
"⚠️ The test run will process **real** open PRs and issues — it is not
a dry run. If you have open items, the agent will attempt to fix them
based on the `fix_mode` you selected in Step 3.

Proceed with the live test run?"

- If the user says yes → execute the gardener runbook once by reading
  `.claude/commands/gardener-manual.md` and following every step.
  This validates that gh auth, context tree access, and PR scanning all work.
- If the user says no → skip the test run and proceed to Step 7.

## 7. Confirm

Output:
"🌱 repo-gardener installed and tested.
- Commands committed and pushed to remote.
- Run `/gardener-start(loop+schedule)` to start automation.
- Run `/gardener-manual` anytime for a one-off run.
- Run `/gardener-stop` to pause everything."
