Start repo-gardener automation for this project.

repo-gardener v2.0 is a context-aware review bot. It comments on PRs
and issues with tree-backed product/context fit analysis. It never
pushes code.

## 1. Verify installed

Check that `.claude/commands/gardener-manual.md` and
`.claude/gardener-config.yaml` both exist locally.

- If either missing → output:
  "❌ repo-gardener is not installed. Run `/gardener-onboarding` first."
  STOP.

## 2. Verify config is on the default branch

The cloud schedule clones the default branch, so the config and command
files must be reachable there — not just on a feature branch.

```bash
current_branch=$(git rev-parse --abbrev-ref HEAD)
remote_url=$(git remote get-url origin 2>/dev/null)
repo_slug=$(echo "$remote_url" | sed -E 's#.*github\.com[:/]([^/]+/[^/.]+).*#\1#')
default_branch=$(gh api "/repos/$repo_slug" --jq .default_branch)

# Does the config exist on the default branch?
gh api "/repos/$repo_slug/contents/.claude/gardener-config.yaml?ref=$default_branch" \
  --silent 2>/dev/null
config_on_default=$?
```

- If `config_on_default == 0` → good, skip to Step 3.
- If `config_on_default != 0` → config is not on default branch.
  Ask the user interactively:

  "⚠️ `.claude/gardener-config.yaml` is not on the remote's default branch
  (`$default_branch`). You're currently on `$current_branch`. The cloud
  schedule can only read files from the default branch.

  Options:
  1. **Merge my feature branch into `$default_branch` first**, then retry.
     (Best for team repos with protected default branches.)
  2. **Start loop only** — skip the cloud schedule for now. Local loop
     works from any branch. You can start the schedule later via
     `/gardener-start` once the branch is merged.
  3. **Open a PR for me** — create a PR from `$current_branch` into
     `$default_branch` with the gardener install commit, then stop so
     I can merge it.

  Which?"

  - Option 1 → STOP, wait for user to merge.
  - Option 2 → set `SKIP_SCHEDULE=true`, proceed.
  - Option 3 → run
    `gh pr create --base $default_branch --head $current_branch --title "chore: install repo-gardener" --body "Installs repo-gardener commands and config so the cloud schedule can read them from the default branch."`
    then STOP.

## 3. Start cloud schedule (unless SKIP_SCHEDULE=true)

Use `RemoteTrigger` tool with `action: "list"`. Look for a trigger named
`repo-gardener`.

- Found and enabled → skip. Log: "Schedule already running."
- Found and disabled → `RemoteTrigger` `action: "update"`,
  `body: {"enabled": true}`.
- Not found → `RemoteTrigger` `action: "create"` with:
  - `name`: "repo-gardener"
  - `cron_expression`: "0 * * * *" (every hour)
  - Prompt: contents of `.claude/commands/gardener-schedule.md`
  - Source repo: the current repo's GitHub URL

## 4. Start local loop

Start: `/loop 10m /gardener-loop`

## 5. Confirm

Output:
"🌱 repo-gardener is running.
- Cloud schedule: <every hour | SKIPPED — config not on default branch>
- Local loop: every 10min (runs while you're here)
- Stop anytime: `/gardener-stop`"
