<!-- Context: this runs locally via /loop on the user's machine -->

**UNATTENDED=true** — you are running under `/loop`. Do not prompt the
user. If a fix is ambiguous or requires human judgment, skip that PR
and log the reason.

**RUN_MODE=loop**

You MUST read `.claude/commands/gardener-respond-manual.md` in this
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
local `/loop 30m /gardener-respond-loop` is armed so this command
keeps re-firing every 30 minutes without the user having to type
`/loop`.

**Idempotency**: if a `/loop` is already armed for `/gardener-respond-loop`,
do NOT arm a second one (would cause double-firing). Call the harness
loop-list facility (e.g. via the `loop` skill's list/status capability,
or `ScheduleWakeup`-tracked state) and skip arming if one already
exists.

If no loop is armed for `/gardener-respond-loop` yet:
- Invoke `/loop 30m /gardener-respond-loop` to arm it.
- Log: `🔁 Armed /loop 30m /gardener-respond-loop`.

If a loop is already armed:
- Log: `🔁 /loop already armed for /gardener-respond-loop, skipping`.

To stop the recurrence, run `/gardener-respond-stop`. The user does
NOT need to manually run `/loop 30m /gardener-respond-loop` — calling
`/gardener-respond-loop` once is enough to start the cron.
