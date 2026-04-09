Start repo-gardener automation for this project.

## 1. Verify installed

Check that `.claude/commands/gardener-manual.md` exists.

- If NOT found → output:
  "❌ repo-gardener is not installed. Run `/gardener-onboarding` first."
  STOP. Do not continue.

## 2. Start cloud schedule

Set up: `/schedule every hour /gardener-schedule`

## 3. Start local loop

Start: `/loop 10m /gardener-loop`

## 4. Confirm

Output:
"🌱 repo-gardener is running.
- Cloud schedule: every hour (runs when your machine is off)
- Local loop: every 10min (runs while you're here)
- Stop anytime: `/gardener-stop`"
