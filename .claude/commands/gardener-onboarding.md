Set up repo-gardener for this project. Do every step in order.

repo-gardener is a **context-aware review bot** — it comments on PRs
and issues with product/context fit analysis, reading from your
project's context tree. It does NOT push code or open PRs.

## 0. Verify repo

Check if you are inside a git repository (`git rev-parse --show-toplevel`).

- If NOT inside a repo → ask the user: "Which repo would you like to
  set up repo-gardener for? Please specify a path or GitHub URL."
  Wait for their answer. cd into that repo before continuing.

## 1. Verify context tree

Search CLAUDE.md and AGENTS.md for a GitHub URL pointing to a context
tree repo (pattern: `github.com/<org>/<repo>` with "tree", "context",
"session", or "memory" in surrounding context).

- If NOT found → ask the user:
  "🌳 No context tree found in this repo. repo-gardener requires a
   context tree to make context-aware reviews.

   Set up a First-Tree for this repo:
   https://github.com/agent-team-foundation/first-tree

   Run `first-tree init` then re-run `/gardener-onboarding`."
  STOP. Do not continue.

## 2. Install gardener commands

Fetch from a release tag for integrity and reproducibility.

```bash
GARDENER_VERSION="v2.0.0"
BASE="https://raw.githubusercontent.com/agent-team-foundation/repo-gardener/${GARDENER_VERSION}"
mkdir -p .claude/commands
curl -sL -o .claude/commands/gardener-manual.md "${BASE}/.claude/commands/gardener-manual.md"
curl -sL -o .claude/commands/gardener-schedule.md "${BASE}/.claude/commands/gardener-schedule.md"
curl -sL -o .claude/commands/gardener-start.md "${BASE}/.claude/commands/gardener-start.md"
curl -sL -o .claude/commands/gardener-loop.md "${BASE}/.claude/commands/gardener-loop.md"
curl -sL -o .claude/commands/gardener-stop.md "${BASE}/.claude/commands/gardener-stop.md"
curl -sL -o .claude/commands/gardener-onboarding.md "${BASE}/.claude/commands/gardener-onboarding.md"
```

Verify downloads are not empty:

```bash
for f in .claude/commands/gardener-*.md; do
  if [ ! -s "$f" ]; then
    echo "❌ Download failed: $f is empty. Check your network and try again."
    exit 1
  fi
done
```

## 3. Determine review mode

Ask the user which mode repo-gardener should run in. Run
`gh repo view --json nameWithOwner,isFork,parent,viewerPermission` to
detect the repo state, then:

1. **Not a fork** → `target_repo=<current>`, `review_mode=maintainer`.
   No prompt needed.

2. **Fork + user has WRITE/MAINTAIN/ADMIN on parent** → user maintains
   upstream. `target_repo=<parent>`, `review_mode=maintainer`. No prompt.

3. **Fork + no upstream write access** → STOP and ask:

   "🔀 This is a fork of `<parent>`. Which mode?

   1. **Maintainer mode** on my fork `<fork>` — review PRs/issues in my fork
   2. **Advisor mode** on upstream `<parent>` — comment on upstream
      PRs/issues with tree-backed context reviews
      (no write access needed — you can comment on any public repo)
   3. **Exit** — run gardener in a different repo

   Which would you like?"

   Wait for the answer.
   - Option 1 → `target_repo=<fork>`, `review_mode=maintainer`
   - Option 2 → `target_repo=<parent>`, `review_mode=advisor`
   - Option 3 → exit cleanly

Write the chosen values to `.claude/gardener-config.yaml`:

```yaml
target_repo: <owner>/<name>
review_mode: maintainer   # or advisor
```

## 4. Commit and push

The command files and config must be in the remote repo so that
`/schedule` (which runs in the cloud) can access them.

```bash
git add .claude/commands/gardener-*.md .claude/gardener-config.yaml
git diff --cached --quiet && echo "No changes to commit" || git commit -m "chore: install repo-gardener v2.0.0"
git push
```

If push fails (e.g. branch protection), tell the user:
"Push failed — your branch may have protection rules. Create a PR
with these changes instead, or push to a different branch."

## 5. Test run

Ask the user:
"⚠️ The test run will scan real open PRs and issues on `<target_repo>`
and post **real comments**. In maintainer mode, comments appear on your
own repo. In advisor mode, comments appear on the upstream repo.

Proceed with the live test run?"

- If yes → execute `.claude/commands/gardener-manual.md` once.
- If no → skip the test run and proceed to Step 6.

## 6. Confirm

Output:
"🌱 repo-gardener v2.0.0 installed.
- Review mode: `<maintainer|advisor>`
- Target repo: `<target_repo>`
- Commands committed and pushed to remote.

**Important**: restart Claude Code (or start a new session) so the new
slash commands are picked up by the registry. After restarting:

- Run `/gardener-start` to start automation (loop + schedule).
- Run `/gardener-manual` for a one-off review.
- Run `/gardener-stop` to pause everything."
