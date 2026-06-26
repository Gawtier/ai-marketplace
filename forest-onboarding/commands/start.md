---
description: Start the Forest Admin headless onboarding (dev or ops flow)
argument-hint: [dev | ops | <use-case description>]
---

Launch the **forest-onboard** skill and drive the headless Forest Admin onboarding end-to-end.

Treat `$ARGUMENTS` as the initial intent:
- `dev` → the user has a database and wants to build & deploy (real-DB flow).
- `ops` → no database, just explore (zero-DB demo flow via `projects:create:demo`).
- anything else → a free-text use-case; infer the flow (no reachable DB ⇒ ops/demo).

Then follow the skill exactly:
1. Run the **preflight** (remediate missing tools / auth; route "no DB" to the ops flow — never fail-fast on it).
2. Collect remaining intent as **arguments** (never a wizard); for the dev flow, get the DB connection via a **`.env` the user writes** — never ask for the URI in chat, never echo a secret.
3. Show the **Plan** recap and get acknowledgement before the first mutating command.
4. Execute the validated flow, honoring **GATE 1** (boot before layout/env), **GATE 2** (deploy before invite), and the 🟦 checkpoints (prod env, deploy, invitations).
5. End with the concise output summary.

If `$ARGUMENTS` is empty, ask the user which flow fits (dev vs ops) before starting.
