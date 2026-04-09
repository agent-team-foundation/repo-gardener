Start repo-gardener automation for this project.

repo-gardener v2.0.0 is a context-aware review bot. It comments on PRs
and issues with tree-backed product/context fit analysis. It never
pushes code.

## 1. Verify installed and configured

Check that `.claude/commands/gardener-manual.md` exists.

- If NOT found → output:
  "❌ repo-gardener is not installed. Run `/gardener-onboarding` first."
  STOP. Do not continue.

Check that `.claude/gardener-config.yaml` exists.

- If NOT found → output:
  "❌ repo-gardener is not configured. Run `/gardener-onboarding` to
   choose a target repo and review mode."
  STOP. Do not continue.

Verify the config has been committed and pushed to the remote. If the
config file is uncommitted or unpushed, remind the user:
  "⚠️ `.claude/gardener-config.yaml` is not pushed to the remote.
   The cloud schedule won't be able to read it. Commit and push first."
  STOP. Do not continue.

## 2. Start cloud schedule

Check if a gardener schedule already exists:
Use the `RemoteTrigger` tool with `action: "list"`. Look for any trigger
whose name contains "gardener".

- If found and enabled → skip, already running. Log: "Schedule already exists."
- If found and disabled → re-enable it with:
  `RemoteTrigger` `action: "update"`, `trigger_id: "<id>"`, `body: {"enabled": true}`
- If not found → create one with:
  `RemoteTrigger` `action: "create"` with:
  - `name`: "repo-gardener"
  - `cron_expression`: "0 * * * *" (every hour)
  - Prompt: the contents of `.claude/commands/gardener-schedule.md`
  - Source repo: the current repo's GitHub URL (from `gh repo view --json url`)

## 3. Start local loop

Start: `/loop 10m /gardener-loop`

## 4. Confirm

Output:
"🌱 repo-gardener is running.
- Cloud schedule: every hour (runs when your machine is off)
- Local loop: every 10min (runs while you're here)
- Stop anytime: `/gardener-stop`"
