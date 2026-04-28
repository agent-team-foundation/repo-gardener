<!-- Context: this runs locally via /loop on the user's machine -->

**UNATTENDED=true** — you are running under `/loop`. Do not prompt the
user. If the runbook requires interactive input (e.g. missing config,
missing bindings), exit with an error log and wait for the user to
fix the issue and restart.

**RUN_MODE=loop** — record this in log output so the user knows
whether a run came from the local loop or the cloud schedule.

You MUST read `.claude/commands/gardener-sync-schedule.md` in this
repo and execute it as a strict runbook. Follow every step sequentially.
For each step:
1. Execute the step
2. Log what you did to stdout
3. Move to the next step

Do not skip steps. Do not summarize. Do not stop early unless a step
explicitly says to exit or stop. If a step fails, follow the step's
own failure instructions (which may include stopping). If no failure
instructions exist, log the error and continue to the next step.

## Self-arming recurrence

After the runbook finishes (success or non-fatal error), ensure a
local `/loop 10m /gardener-sync-loop` is armed so this command keeps
re-firing every 10 minutes without the user having to type `/loop`.

**Idempotency**: if a `/loop` is already armed for `/gardener-sync-loop`,
do NOT arm a second one (would cause double-firing). Call the harness
loop-list facility (e.g. via the `loop` skill's list/status capability,
or `ScheduleWakeup`-tracked state) and skip arming if one already
exists.

If no loop is armed for `/gardener-sync-loop` yet:
- Invoke `/loop 10m /gardener-sync-loop` to arm it.
- Log: `🔁 Armed /loop 10m /gardener-sync-loop`.

If a loop is already armed:
- Log: `🔁 /loop already armed for /gardener-sync-loop, skipping`.

To stop the recurrence, run `/gardener-sync-stop`. The user does NOT
need to manually run `/loop 10m /gardener-sync-loop` — calling
`/gardener-sync-loop` once is enough to start the cron.
