---
name: forest-onboard
description: >
  Onboard onto Forest Admin end-to-end, headlessly (no UI), Standalone-first.
  Use this skill when someone wants to "set up Forest Admin from scratch",
  "create a Forest Admin project and admin panel", "onboard onto Forest Admin
  without the wizard", "just see Forest Admin running" (zero-DB demo), or go all
  the way from login to a deployed production admin panel with an invited team.
  It orchestrates the CLI (toolbelt) and the sibling skills (boot-standalone-agent,
  deploy-heroku) and hands off customizations to the forest-code skill.
---

# Forest Admin — headless onboarding (orchestrator)

Drive the whole Standalone onboarding from the terminal, in the **validated order**, owning waits, errors and the human checkpoints. You collect intent as **arguments** — never run a wizard.

Principle: **skills decide, the CLI executes.** Every step is a thin layer over a deterministic `forest` command.

## STOP — preflight & "show before act" (before any mutation)

1. **Pick the flow first** (see *Two flows* below) — it decides what preflight even checks. Routing rule: **has a reachable DB → dev flow; no DB → ops/demo flow.**
2. **Preflight, scoped to the chosen flow.** Three reactions:
   - 🟩 **REMEDIATE** then continue: a missing tool (`node`/`npm` or ruby, `forest`, `git`, and `heroku` *only if deploying*), **not authenticated**, or a missing input you can ask for. Install / guide / ask, then re-check.
     - **Auth:** check with **`forest user`** (prints the logged-in email when a session exists). There is **no `forest signup`** — account creation stays in the web UI. If `forest user` shows no session: run `forest login`; if the user has no account, point them to **https://app.forestadmin.com** to sign up (email verification + ToS happen there), then `forest login`.
   - 🟥 **FAIL-FAST**: only the genuine non-recoverables — empty/invalid schema after connecting, readiness timeout, deploy build failure, prod never active. **"No database" is NOT a fail-fast** — it routes to the **ops/demo flow** (zero-DB). Only fail-fast on the DB if the user is committed to the dev flow *and* their chosen DB is unreachable.
   - 🟦 **CHECKPOINT** (ask first) before any outward/destructive action: creating the production env, deploying, and **sending invitations** (real emails go out).
3. **Show a "Plan" recap** (chosen flow + collected intent + which steps will run) and get acknowledgement before the first mutating command.

## Secrets & credentials (never echo to the model)

The most dangerous secret here is the **database connection URI** — it carries your real DB password (full data access). `FOREST_ENV_SECRET`/`FOREST_AUTH_SECRET` are lower-risk (Forest-scoped, tool-generated).

**Hard rule: a secret must never enter the model's token stream** (everything Claude emits goes to provider logs + transcript). Secrets travel **by reference, never by value**:

- **DB URI → `.env` first.** Before creating anything in the dev flow, have the **user** write the connection into a `.env`:
  ```
  DATABASE_URL=postgres://…
  DATABASE_SCHEMA=public
  DATABASE_SSL_MODE=required
  ```
  Then source it and pass it by reference: `set -a; . ./.env; set +a` → `forest projects:create:sql -c "$DATABASE_URL" …`. The shell expands `$DATABASE_URL`; Claude only ever emits the **variable name**.
- **Never** ask the user to paste the URI into the chat, **never** `cat`/print a `.env`, **never** put a secret in a command literal or in the final recap.
- **Reading a secret back** (e.g. the prod `secretKey`): pipe it, don't print it — `forest environments:get <id> --format json | jq -r .secretKey | …` straight into `heroku config:set`, so the value never lands in the model's context.
- Fallback channel: the CLI's own **interactive prompt** (user types into their TTY) — even safer than env vars (no argv/`ps` exposure either).
- A hand-rolled root `.env` **must be gitignored** (else it ships via `git push heroku`).

## Two flows (choose at the start)

Frame the onboarding around **who the user is and whether they have a database** — not a technical toggle. Two flows:

- **Dev flow — "I have a database, I want to build & deploy."** A developer connecting Forest to their own data → `forest projects:create:sql` / `:nosql`. Goes the full distance: real DB → boot → layout → customizations → production → deploy → invite. **The dev DB and the prod DB can differ** (see below).
- **Ops flow — "I have no database, I want to explore."** Someone evaluating Forest, no DB / no Docker → `forest projects:create:demo` scaffolds an agent on a **self-contained fintech demo datasource** + ships a curated layout file. Boot → apply layout → explore locally. Data is ephemeral; deploy is possible but rarely the point.

**Routing rule: no reachable database ⇒ ops/demo flow** (never block on "no DB"). If the user has a DB and wants their real data, it's the dev flow. Ask which fits; if they're unsure or DB-less, start the ops flow.

> These mirror the two Linear projects: *Headless onboarding · CLI & orchestration* (dev) and *Headless onboarding for ops* (ops).

### Ops flow — defaults (do NOT ask, just proceed)

The ops persona **explores**; they are not a developer. Minimise questions and jargon:
- **Language**: always **TypeScript** — never ask.
- **No production, no deploy, no team invites** — never even bring them up. Ops = explore locally and stop.
- **Project name**: default to a **unique** name `forest-demo-<short random suffix>` (e.g. `forest-demo-7f3k`) to dodge collisions with reserved/leftover names from a retry. Only use a custom name if the user volunteered one.
- **Everything non-interactive** — pass *all* flags so the CLI never prompts (hostname/port → `localhost`; environment/team → resolved explicitly, see the steps). **If a prompt still appears, you forgot a flag** — find it, don't answer the prompt by hand.
- **Plan recap in plain language** — no "GATE 1 / apimap / schema / dev environment". Say what the user gets, e.g.:
  > 1. Build your demo admin panel (sample fintech data, no setup). 2. Start it. 3. Apply a clean, ready-made layout. 4. Give you the link to open and explore.

## Collect intent (arguments, not a wizard)

- flow (dev / ops) · use-case / app name · language *(dev flow only — ops is always TypeScript)*.
- **(dev flow) dev database** connection — collected via a **`.env` the user writes** (`DATABASE_URL=…`), not pasted in chat (see *Secrets & credentials*). May be a **local** DB (`localhost` / Docker).
- **(dev flow only)** whether to go to **production** (deploy) and on which PaaS (Heroku first).
  - If deploying: the **production database — which may be a different DB than dev.** It **must be remotely reachable** — a local dev DB will NOT work from a PaaS. **Ask** whether prod uses the **same** DB as dev (only valid if dev already uses a remote DB) or a **dedicated prod DB**, and capture that prod `DATABASE_URL` **separately** (its own `.env` / `heroku config`, never the dev value reused blindly). 🟦 If it's the same DB, warn that admin actions hit real production data.
- **(dev flow only)** whether to **invite** teammates (emails, role, permission level, team). `teams:create|delete` exist if you need to set up a team first.

## The validated flow (keep this order — two hard gates)

1. **Account** — check auth with **`forest user`** (it prints the logged-in email). If it shows no session: `forest login`; if the user has no account, sign up in the **web UI** first (see Auth above), then `forest login`. Token persists across commands.
2. **Project + dev env** —
   - Dev flow — **`.env` first** (secret-safe): have the user write `DATABASE_URL`/`DATABASE_SCHEMA`/`DATABASE_SSL_MODE` into a `.env`, then `set -a; . ./.env; set +a` and run `forest projects:create:sql` / `:nosql -c "$DATABASE_URL" -l <lang> -s "$DATABASE_SCHEMA" -H <host> -P <port> --databaseSSL`. Pass the URI **by reference** (`$DATABASE_URL`) — never inline the value. (Or let the CLI prompt interactively.)
   - Ops flow — **fully non-interactive**: `forest projects:create:demo forest-demo-<suffix> -l typescript -H http://localhost -P 3310`. Always pass `-H`/`-P` (localhost) so it **never prompts for hostname**, and a unique `forest-demo-<suffix>` name. No DB, no `.env`.
   - Either way this creates the project, the **dev environment**, the agent scaffold and (dev flow) the project's own `.env`. **Never** use legacy `forest projects create` (forest-express v1).
3. **Boot the dev agent** → use the **`boot-standalone-agent`** skill. The agent pushes its schema → dev env active.
   - 🚧 **GATE 1**: an environment must have pushed its apimap here, otherwise later steps are refused (*"Please finalize the configuration…"*).
4. **Configure the layout** — make the panel readable. The layout is UI-only and **does not require redeploy** (it is pushed to Forest, not your code).
   - 🚧 **Boot first (GATE 1).** `forest layout:apply` is **rejected until the agent has booted and pushed its schema** — Forest needs an existing rendering to patch. So the order is always **create → boot (step 3, `npm start`) → layout:apply**. Never apply a layout on a freshly-created env that hasn't run yet.
   - Ops flow: `create:demo` **ships a curated `forest-layout.json` in the scaffold but does NOT apply it** — that is on you. After boot, **non-interactively**: `forest layout:apply forest-layout.json -p <projectId> -e <dev env name|id> -f` — pass `-p`/`-e` (resolve them from the create step / `forest environments:list -p <projectId> --format json`) to skip the project/environment prompts, and `-f` to skip the confirmation. (Don't skip this step — without it the demo panel is uncurated despite the bundled file.)
   - Dev flow: after boot, `forest layout:pull` → edit `forest-layout.json` (hide technical fields `*_id`/`passwordHash`/timestamps, set sensible record titles, group fields) → `forest layout:apply forest-layout.json`.
5. **Customizations** *(optional)* → hand off to the **`forest-code`** skill (actions, fields, hooks, segments). After any customization, **regenerate `.forestadmin-schema.json` + commit + redeploy** (production reads the frozen schema).
6. **Production environment** *(optional)* — `forest environments:create --type production -n Production` (URL may be omitted; created **inactive**, **no role yet**).
7. **Deploy** *(optional)* → use the **`deploy-heroku`** skill: push the agent code with the **production** `FOREST_ENV_SECRET`. The agent pushes its schema to prod → `apimapVersionId` **+ the first role ("Operations") is created here**. Then set the env's `apiEndpoint` (`forest environments:update -e <id> -u <url>`) → `isActive: true`.
   - 🚧 **GATE 2**: the **first role is created by this deployment** — so **inviting before deploying fails** ("No role found"). Either deploy first, or create a role up front with `forest roles:create -n <name>`.
8. **Invite the team** *(optional)* — `forest users:invite -e <email> -l <level> [-r <role>] [-t <team>]`. `-e` repeats for several users; resolves role/team **by name**. 🟦 sends real emails.

## Output (end of run)

- **Ops flow** — keep it plain and end on the link. Print the **panel URL** `https://app.forestadmin.com/<project-name>` with a one-liner like *"Your demo admin panel is live — open this to explore."* No env-vars block, no technical ids, no "next steps" jargon.
- **Dev flow** — a concise summary: projectId, dev (& prod) environment ids, the **production env-vars block** (`FOREST_ENV_SECRET`, generated `FOREST_AUTH_SECRET`, `DATABASE_URL`) when deployed, what is live, and any remaining manual steps. The panel is at `https://app.forestadmin.com/<project-name>`.

## Notes

- **Readiness**: local = the agent log line `Successfully mounted on Standalone server`; remote = poll `forest environments:get … --format json` (`isActive = apimapVersionId && apiEndpoint`).
- **CLI surface**: `login`, `projects:create:demo|sql|nosql`, `environments:create|update|delete|reset|get`, `users:invite|edit|list`, `roles:create|apply|copy|export`, `layout:pull|apply`, `schema:update|apply|diff`, and `teams:create|delete|copy-layout` are all in the toolbelt (`main`). `teams:*` merged recently, so a slightly older published `forest-cli` may not expose them yet — if `forest teams` errors with "Command not found", update the CLI (`npm i -g forest-cli`). **Not in the CLI at all:** `signup` (account creation stays in the web UI).
- Keep the visibility rule from the in-app skill: **show the detected stack before acting**.
- Concepts grounding: see `references/concepts.md` for the minimal vocabulary (environment types, agent, schema↔apimap, role/team/permission, layout, demo datasource).

## Companion plugins (auto-installed, degrade gracefully)

This plugin **declares `forest-code` and `forest-docs` as dependencies** (see `plugin.json`), so on a modern Claude Code they are installed alongside it — the handoff is seamless. If a companion is somehow absent, degrade gracefully.

| Need | Use | If not installed |
|---|---|---|
| **Write customizations** (actions, fields, hooks, segments…) | **`forest-code`** plugin (handles modern `@forestadmin/agent`; it self-guards legacy) | Recommend installing `forest-code`; do not hand-roll customization code. |
| **Product questions** ("what is X", how a feature works, any topic) | **`forest-docs`** plugin (MCP → live docs, search + read) | Answer from `references/concepts.md`; point to https://docs.forestadmin.com. |

After any `forest-code` customization, run the **redeploy loop** (regenerate `.forestadmin-schema.json` → commit → redeploy) — see the `deploy-heroku` skill. `forest-code` writes the code; **it does not redeploy** — that is this plugin's job.
