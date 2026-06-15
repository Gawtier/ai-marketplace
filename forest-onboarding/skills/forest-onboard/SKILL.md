---
name: forest-onboard
description: >
  Onboard a developer onto Forest Admin end-to-end, headlessly (no UI),
  Standalone-first. Use this skill when someone wants to "set up Forest Admin
  from scratch", "create a Forest Admin project and admin panel", "onboard onto
  Forest Admin without the wizard", or go all the way from signup to a deployed
  production admin panel with an invited team. It orchestrates the CLI
  (toolbelt) and the sibling skills (boot-standalone-agent, deploy-heroku) and
  hands off customizations to the forest-code skill.
---

# Forest Admin — headless onboarding (orchestrator)

Drive the whole Standalone onboarding from the terminal, in the **validated order**, owning waits, errors and the human checkpoints. You collect intent as **arguments** — never run a wizard.

## STOP — preflight & "show before act" (before any mutation)

1. **Preflight, scoped to the chosen path.** Check only what the path needs. Three reactions:
   - 🟩 **REMEDIATE** then continue: a missing tool (`node`/`npm` or ruby, `forest`, `git`, and `heroku` *only if deploying*), not authenticated, or a missing input you can ask for. Install / guide / ask, then re-check.
   - 🟥 **FAIL-FAST**: **no reachable database** (the "no DB" path is out of scope for now); and runtime non-recoverables later (empty/invalid schema, readiness timeout, deploy build failure, prod never active).
   - 🟦 **CHECKPOINT** (ask first) before any outward/destructive action: creating the production env, deploying, and **sending invitations** (real emails go out).
2. **Show a "Plan" recap** (collected intent + which steps will run) and get acknowledgement before the first mutating command.

## Collect intent (arguments, not a wizard)

- use-case / app name · language (`javascript`/`typescript`)
- **dev database** connection (URL) — may be a **local** DB (`localhost` / Docker).
- whether to go to **production** (deploy) and on which PaaS (Heroku first).
  - If deploying: the **production database**. It **must be remotely reachable** — a local dev DB will NOT work from a PaaS. **Ask** whether prod uses the **same** DB as dev (only valid if dev already uses a remote DB) or a **dedicated prod DB**, and capture that prod `DATABASE_URL`. 🟦 If it's the same DB, warn that admin actions hit real production data.
- whether to **invite** teammates (emails, team, role, permission level).

## The validated flow (keep this order — two hard gates)

1. **Account** — `forest signup` (or `forest login` if it exists). Token persists like `login`.
2. **Project + dev env** — `forest projects:create:sql` / `:nosql` with non-interactive flags (`-c <uri> -l <lang> -s <schema> -H <host> -P <port> --databaseSslMode …`). Creates the project, the **dev environment**, the agent scaffold and `.env`. **Never** use legacy `forest projects create`.
3. **Boot the dev agent** → use the **`boot-standalone-agent`** skill. The agent pushes its schema → dev env active.
   - 🚧 **GATE 1**: an environment must have pushed its apimap here, otherwise step 4 is refused (*"Please finalize the configuration…"*).
4. *(optional)* **Customizations** → hand off to the **`forest-code`** skill. After any customization, **regenerate `.forestadmin-schema.json` + commit + redeploy** (production reads the frozen schema).
5. **Production environment** — `forest environments:create --type production` (no URL needed; created **inactive**, **no role yet**).
6. **Deploy** → use the **`deploy-heroku`** skill: push the agent code with the **production** `FOREST_ENV_SECRET`. The agent pushes its schema to prod → `apimapVersionId` **+ the first role ("Operations") is created here**. Then set the env's `apiEndpoint` (`forest environments:update -u <url>`) → `isActive: true`.
   - 🚧 **GATE 2**: the **first role is created by this deployment** — so **inviting before deploying fails** ("No role found"). Either deploy first, or create a role via `POST /api/roles`.
7. **Invite the team** *(optional)* — `forest users:invite -e <email> --team <name> --role <name> -l <level>`. Resolves team & role **by name**.

## Output (end of run)

Return a concise summary: projectId, dev & prod environment ids, the **production env-vars block** (`FOREST_ENV_SECRET`, generated `FOREST_AUTH_SECRET`, `DATABASE_URL`), what is live, and any remaining manual steps.

## Notes

- Readiness: local = the agent log line `Successfully mounted on Standalone server`; remote = poll `forest environments:get … --format json` (`isActive = apimapVersionId && apiEndpoint`).
- Some CLI commands ship via toolbelt PRs (#768 signup, #769 environments, #770 users:invite) — check availability and fall back to the documented API endpoints if a command is missing.
- Keep the visibility rule from the in-app skill: **show the detected stack before acting**.
- Concepts grounding: see `references/concepts.md` for the minimal vocabulary (environment types, agent, schema↔apimap, role/team/permission, layout).

## Companion plugins (recommended, not bundled)

This plugin orchestrates the flow; it **delegates** to two sibling plugins for jobs it does not do itself. They are **separate, optional installs** — recommend them, and **degrade gracefully** if absent.

| Need | Use | If not installed |
|---|---|---|
| **Write customizations** (actions, fields, hooks, segments…) | **`forest-code`** plugin (handles modern `@forestadmin/agent`; it self-guards legacy) | Tell the user to install `forest-code`; do not hand-roll customization code. |
| **Product questions** ("what is X", how a feature works, any topic) | **`forest-docs`** plugin (MCP → live docs, search + read) | Answer from `references/concepts.md`; point to https://docs.forestadmin.com. |

After any `forest-code` customization, run the **redeploy loop** (regenerate `.forestadmin-schema.json` → commit → redeploy) — see the `deploy-heroku` skill. `forest-code` writes the code; **it does not redeploy** — that is this plugin's job.
