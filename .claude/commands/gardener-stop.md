Stop all repo-gardener automation for this project.

## 1. Stop local loop

If a `/loop` is currently running `/gardener-loop`, stop it.

## 2. Disable cloud schedule

List all scheduled triggers with the RemoteTrigger tool (`action: "list"`).
Find any trigger whose name is `repo-gardener`.
For each one, disable it with the RemoteTrigger tool:
`action: "update"`, `trigger_id: "<id>"`, `body: {"enabled": false}`

Do NOT delete the trigger — just disable it so it can be re-enabled later.

## 3. Confirm

Output:
"🌱 repo-gardener stopped.
- Local loop: stopped
- Cloud schedule: disabled (not deleted)
- Restart anytime: `/gardener-start(loop+schedule)`"
