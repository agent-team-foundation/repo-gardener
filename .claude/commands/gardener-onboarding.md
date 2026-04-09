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

## 1. Install gardener commands

Fetch from a pinned release tag for integrity.

```bash
GARDENER_VERSION="v2.0.1"
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

## 2. Choose target repo

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

(Config is written at the end of Step 3, after the tree URL is resolved.)

## 3. Find the context tree

Try these sources in order, picking the first match:

1. **Target repo's CLAUDE.md / AGENTS.md** (if `target_repo` is the
   current repo, grep local files; if it's a different repo, use
   `gh api /repos/$target_repo/contents/CLAUDE.md` and
   `gh api /repos/$target_repo/contents/AGENTS.md` — decode base64).
2. **User's global `~/CLAUDE.md`** (`cat ~/CLAUDE.md 2>/dev/null`).
3. **Ask the user**.

For sources 1 and 2, use this regex to extract GitHub URLs:

```bash
grep -oE '(https://)?github\.com/[a-zA-Z0-9._-]+/[a-zA-Z0-9._-]+'
```

Prefer URLs where the surrounding ~200 characters contain one of:
`tree`, `context`, `memory`, `session`, `first-tree`, `kael-tree`.

**Always confirm the URL with the user** before saving — even when
auto-detected — to avoid false positives. Show the detected URL and ask
"Use this as your context tree? (y/n/different)".

**If no URL is found or the user says "different"**, ask:

"🌳 No context tree found automatically. Gardener needs a tree URL to
do context-aware reviews. Choose one:

1. **Enter a URL** — paste a `github.com/<org>/<repo>` URL
2. **Set up a new First-Tree** — https://github.com/agent-team-foundation/first-tree
   (run `first-tree init`, then re-run `/gardener-onboarding`)
3. **Skip** — install gardener without a tree; reviews will be limited
   until you add one via `.claude/gardener-config.yaml`"

- Option 1 → validate the URL with `gh repo view <owner>/<name>`,
  set `tree_repo=<url>`
- Option 2 → STOP
- Option 3 → set `tree_repo=""` (gardener will mark all verdicts as
  `INSUFFICIENT_CONTEXT` until the tree is set)

Write the config (now includes tree_repo):

```bash
cat > .claude/gardener-config.yaml <<EOF
target_repo: <owner>/<name>
tree_repo: <tree-url-or-empty>
EOF
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
