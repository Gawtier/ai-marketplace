---
name: boot-standalone-agent
description: >
  Install dependencies and boot a freshly scaffolded Forest Admin Standalone
  agent locally so it pushes its schema and the development environment becomes
  active. Use right after `forest projects:create:sql/nosql/demo`, when someone
  wants to "start my Forest Admin agent", "run the agent locally", or "see my
  data in Forest Admin". Pairs with forest-onboard.
---

# Boot a Standalone agent (dev)

Brings a scaffolded agent up locally so it pushes its schema → the **dev environment becomes active**. This also satisfies the gate required before creating other environments ("finalize configuration"). Works the same for **real-DB** (`create:sql/nosql`) and **demo** (`create:demo`) scaffolds — the demo just has an in-memory datasource and no `.env`/`DATABASE_URL`.

## Prerequisites (preflight)

- 🟩 REMEDIATE if missing: `node`/`npm` (or ruby), and the generated project directory (with its `.env` for the real-DB scaffold; the demo scaffold has none).
- 🟥 FAIL-FAST (**real-DB scaffold only**): the `DATABASE_URL` in `.env` is unreachable (nothing to serve). The **demo scaffold has no database** — skip this check.

## Show before act

Before running anything mutating, show the **detected stack** (framework, ORM/datasource — or "in-memory demo datasource", database, entry point, port) read from the project files, and get acknowledgement.

> 🔒 **Secrets**: the project `.env` holds the `DATABASE_URL`. Reference it, **never `cat`/print it** — its value must not enter the model's context (see the orchestrator's *Secrets & credentials*).

## Procedure

```bash
cd <generated-project-dir>
npm install
npm start
```

> ⚠️ **Demo landmine** — `create:demo`'s bundled install can print "Hooray installation success!" yet leave `node_modules/@forestadmin` **empty**, so `npm start` fails to resolve `@forestadmin/*`. Before booting, **verify** `node_modules/@forestadmin` is populated; if not, **re-run `npm install`** and re-check.

## Readiness signal

Wait for the agent log lines:

```
info: Schema was updated, sending new version (hash: …)
info: Successfully mounted on Standalone server (http://0.0.0.0:<port>)
```

- That means the **apimap was pushed** → the dev env is active. You can stop the process (`Ctrl+C`) afterwards; the apimap is recorded server-side.
- Alternatively confirm via `forest environments:get <dev env id> --format json` (`isActive = apimapVersionId && apiEndpoint`).

## Notes

- Dev environments have **roles disabled** (`areRolesDisabled`), so **no role is created** at this stage — that happens on production deployment (see `deploy-heroku` / `forest-onboard` GATE 2).
- Fail-fast if the agent exits before mounting (print the error: DB connection, missing deps, port already in use).
