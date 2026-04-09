Set up repo-gardener for this project. Do every step in order.

repo-gardener is a **context-aware review bot** — it comments on PRs
and issues with product/context fit analysis, reading from your
project's context tree. It does NOT push code or open PRs.

## 0. Verify repo

Check if you are inside a git repository (`git rev-parse --show-toplevel`).

- If NOT inside a repo → ask the user: "Which local directory should I
  use as the working repo? Please cd there first, then re-run
  `/gardener-onboarding`."
  STOP.

## 1. Verify context tree

Search CLAUDE.md and AGENTS.md for a GitHub URL pointing to a context
tree repo. Use this regex:

```bash
grep -oE '(https://)?github\.com/[a-zA-Z0-9._-]+/[a-zA-Z0-9._-]+' CLAUDE.md AGENTS.md 2>/dev/null
```

Among matches, prefer URLs where the surrounding 200 characters contain
one of: `tree`, `context`, `memory`, `session`, `first-tree`, `kael-tree`.

If multiple candidates → ask the user which one is the tree.

- If NOT found → ask:
  "🌳 No context tree found in this repo. repo-gardener requires a
   context tree to do context-aware reviews.

   Set up a First-Tree:
   https://github.com/agent-team-foundation/first-tree

   Run `first-tree init` then re-run `/gardener-onboarding`."
  STOP.

## 2. Install gardener commands

Fetch from a pinned release tag for integrity.

```bash
GARDENER_VERSION="v2.0.0"
BASE="https://raw.githubusercontent.com/agent-team-foundation/repo-gardener/${GARDENER_VERSION}"
mkdir -p .claude/commands
curl -fsSL -o .claude/commands/gardener-manual.md "${BASE}/.claude/commands/gardener-manual.md"
curl -fsSL -o .claude/commands/gardener-schedule.md "${BASE}/.claude/commands/gardener-schedule.md"
curl -fsSL -o .claude/commands/gardener-start.md "${BASE}/.claude/commands/gardener-start.md"
curl -fsSL -o .claude/commands/gardener-loop.md "${BASE}/.claude/commands/gardener-loop.md"
curl -fsSL -o .claude/commands/gardener-stop.md "${BASE}/.claude/commands/gardener-stop.md"
curl -fsSL -o .claude/commands/gardener-onboarding.md "${BASE}/.claude/commands/gardener-onboarding.md"
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

## 3. Choose target repo

Ask the user which repo gardener should review:

"Which GitHub repo should gardener review?

1. **This repo** (`<current owner/name>`) — useful if you're a maintainer
   reviewing your own team's work.
2. **A different repo** — useful if you're contributing context reviews
   to an external OSS project (any public repo works; gardener comments
   don't require write access).

Which would you like?"

- Option 1 → `target_repo=<current owner/name from gh repo view>`
- Option 2 → ask the user for `owner/name`, validate with:
  ```bash
  gh repo view <owner>/<name> --json nameWithOwner
  ```
  If validation fails → re-ask.

Write the config:

```yaml
# .claude/gardener-config.yaml
target_repo: <owner>/<name>
```

## 4. Add the tree cache directory to .gitignore

```bash
grep -q '.gardener-tree-cache' .gitignore 2>/dev/null || \
  echo '.gardener-tree-cache/' >> .gitignore
```

## 5. Commit and push

The command files and config must be in the current repo so that
`/schedule` (cloud agent) can read them.

```bash
git add .claude/commands/gardener-*.md .claude/gardener-config.yaml .gitignore
git diff --cached --quiet && echo "No changes to commit" || \
  git commit -m "chore: install repo-gardener v2.0.0"
git push
```

If push fails (branch protection), tell the user:
"Push failed — your branch may have protection rules. Create a PR
with these changes instead, or push to a different branch, then
continue onboarding."

## 6. Test run (optional)

Ask the user:
"⚠️ The test run will scan open PRs and issues on `<target_repo>` and
may post real comments. Proceed?"

- Yes → execute `.claude/commands/gardener-manual.md` once.
- No → skip to Step 7.

## 7. Confirm

Output:
"🌱 repo-gardener v2.0.0 installed.
- Target repo: `<target_repo>`
- Commands and config committed and pushed to remote.

**Important**: restart Claude Code (or start a new session) so the new
slash commands are picked up. After restarting:

- Run `/gardener-start` to start automation (loop + schedule).
- Run `/gardener-manual` for a one-off review.
- Run `/gardener-stop` to pause everything."
