---
name: gravi-burners
description: >-
  Deploy and manage Gravitate "burner" ephemeral test environments via the `gravi burner` CLI
  for the supply_and_dispatch_aio monorepo. Use this whenever the user wants to spin up / start /
  refresh / sync / recreate / restart / delete a burner, deploy a branch or PR to a burner so they
  (or you) can test a change in a real running app, grab a burner's URL or DB connection, or
  troubleshoot a burner that won't start, is stale, or expired. Triggers on "burner", "gravi",
  "spin up a test env", "put this on a burner", "deploy the branch to test", "ephemeral
  environment", or any `{id}.burner.gravitate.energy` URL — even when the exact command isn't named.
---

# Gravi Burners

A **burner** is an ephemeral, fully-functional copy of the Gravitate app deployed to a GKE
namespace, reachable at `https://{id}.burner.gravitate.energy`. It's how you test a feature branch
in a real running app (frontend + backend + workers + Mongo/Redis/Kafka) instead of just locally.
Burners auto-expire after their TTL and are deleted.

Burners deploy a **pre-built per-SHA Docker image**, not your local working tree. So the loop is
always: **commit → push → CI builds the image → deploy that build to a burner.** Uncommitted local
changes never reach a burner.

> Burner *lifecycle* lives here. For the rest of the `gravi` CLI (auth, instances, config/creds,
> `gravi query`, `gravi switch`, tokens, pastes, ArgoCD watch), see the **gravi-cli** skill.

## Prerequisites

- `gravi` CLI (installed at `~/.local/bin/gravi`; a `~/.pyenv/shims/gravi` shim points at it). `gravi --help` lists commands; `gravi burner --help` for the burner subcommands. Confirm exact flags there — it's version-matched.
- Authenticated: `gravi login` (browser device-flow). If a command says you're unauthorized, run `gravi login`.
- For build status you need `gh` (GitHub CLI) authenticated against `gravitate-energy/supply_and_dispatch_aio`.

## The deploy flow (branch → burner)

1. **Commit and push the feature branch.** Burners build from a pushed SHA.
2. **CI builds per-SHA images automatically** on push. The relevant GitHub Actions workflows are
   `Backend Workflow`, `Frontend Workflow`, and the per-service `*-workflow.yml` (auth, forecast,
   ims, payroll, pricing, scim, inventory-valuation). They trigger on **every** branch
   (`branches: ["**", "!\\!*"]`) — including `KB-*`.
   > The root `CLAUDE.md` note that CI "excludes KB-*" is **stale** — KB-* branches do build.
3. **Wait for the builds to go green** (at minimum `Backend Workflow` + `Frontend Workflow`; the
   build picker only lets you deploy green builds). Watch one with:
   ```bash
   gh run list --branch <branch> --limit 8
   gh run watch <run-id> --exit-status --interval 20
   ```
   - Backend build runs the test suite (~3–4 min). Frontend build is Vite/esbuild only — **no `tsc` gate** — so stale `frontend/src/types/api/v1.d.ts` (un-regenerated `genapi`) does NOT block a build or runtime; types just resolve to `any`.
   - A backend-only change still rebuilds the frontend image (and vice-versa); if the other side's content is unchanged it's retagged cheaply, not rebuilt.
4. **Start the burner on that build:**
   ```bash
   gravi burner start --build <branch-or-SHA> --ttl 8 --dataset default --sync
   ```
   - `--build` accepts a branch name (latest green build) or a specific SHA. `--skip-build-check` bypasses the client-side "build exists across services" check (rarely needed; usually means you didn't wait for green).
   - `--ttl <hours>` (1–168). `--dataset default` seeds the shared testbed (stores/terminals/etc.). Other seed options: `--sheet-id <id>`, `--dataset-version <n>`, `--source-env <env>`, or omit for a barebone (no-data) burner.
   - `--sync` blocks until ready (~6–7 min for a full seed). Without `--sync` it returns immediately with the `burner_id`; poll `gravi burner status <id>` until `Status: ready`.
   - The URL is `https://{burner_id}.burner.gravitate.energy`.

## Command reference

```bash
gravi burner start --build <branch> --ttl 8 --dataset default [--sync] [--json]
gravi burner list
gravi burner status <id>                 # Status / URL / TTL / build / seed
gravi burner delete <id>
gravi burner recreate <id> [--sync]      # full redeploy w/ latest builds + re-seed (see below)
gravi burner autosync trigger <id>       # roll deployments onto rebuilt images IN PLACE (see below)
gravi burner autosync enable|disable|status <id>
gravi burner restart <id> <service>      # rolling-restart ONE deployment's pods in place (e.g. backend)
gravi burner logs <id> <service>         # e.g. backend
gravi burner pods <id>                   # pod readiness per service
gravi burner duplicate <id>              # clone settings to a new namespace
gravi config <id> [--format json|env]    # url / conn_str / dbs / status / expires_at
```

> **No `extend` command.** TTL can only be set at `start` (`--ttl`, 1–168h) — there is no
> `gravi burner extend`. Set a generous TTL up front; if a burner is near expiry, `start`/`recreate`
> a fresh one (you get a new id/URL).

## Refreshing a running burner after a new push (the key decision)

You pushed a fix and the new image is green. How you get it onto an existing burner depends on
**what changed** — this distinction matters and is easy to get wrong:

- **`gravi burner autosync trigger <id>`** — rolls the burner's deployments onto the rebuilt
  images **in place**. Fast, and the **MongoDB data is preserved** (just the pods restart). It only
  swaps images whose tag actually changed (a backend-only push won't roll the frontend, etc.).
  **Use this for image-only changes** — frontend changes, or backend app/endpoint code.
  It does **NOT** re-run `deployment_main`.

- **`gravi burner recreate <id>`** — tears down and **fully redeploys** with the latest builds and
  **re-seeds** the data, keeping the same id/URL. Use this when the change is in
  **`deployment_main`** (the deploy routine in `backend/deployment_script.py`: DB index
  maintenance, migrations, default-data setup). Those run **only on a full deploy** — app
  startup/lifespan only runs cache warmup (`warm_caches`), **not** `deployment_main` — so a plain
  `autosync` won't apply an index/migration change. Caveat: `recreate` has been seen to **flake**
  (abort right after name allocation, leaving the burner `deleted`). If it fails, just
  `gravi burner start` a fresh one — `start` is more reliable; you get a new id/URL.

- **`gravi burner restart <id> <service>`** — rolling-restarts just that one deployment's pods
  **without changing images or data**. Use it to clear a stuck/unhealthy pod or force a re-read of
  mounted config — **not** to pick up a new build (that's `autosync`).

Rule of thumb: **code/UI change → `autosync trigger`; index/migration/seed change → `recreate`
(or a fresh `start`); stuck pod → `restart`.**

## Operational gotchas

- **Data is on the shared dev Atlas cluster.** `gravi config <id>` returns `conn_str` + `dbs`; the
  per-burner databases are named `burner_<id>_<service>` (e.g. `burner_truck_backend`). There are
  **no plain login credentials** in the config — seeded users come from the dataset's Users tab.
  (Tip: `gravi query <id> <db> <collection>` reads burner Mongo directly, no tunnel — see gravi-cli.)
- **Base seed ≠ transactional data.** A fresh burner gets config (locations, stores, products,
  drivers) but **routes / orders / supply plans are generated by activity** (running a model,
  using the app), not by the base seed — they can appear a few minutes after `ready` or only once
  you exercise the relevant flow. Don't assume an empty grid means a broken seed.
- **TTL expiry deletes the burner.** Set a generous `--ttl` at `start`; there's no `extend`, so if
  `status` returns `deleted` / "not ready" it likely expired — `start`/`recreate` a fresh one.
- **All pods `Running` but `Status: failed`** usually means the seed/deploy orchestration failed,
  not the app. Check `gravi burner pods <id>` and `gravi burner logs <id> backend`; often a fresh
  `start` is the fastest recovery.
- **Hard-refresh the browser** (Ctrl/Cmd-Shift-R) after a frontend sync to pick up the new bundle.

## End-to-end example

```bash
# 1. ship the fix
git add -A && git commit -m "KB-XXXXX: ..." && git push

# 2. wait for the image to build green
gh run list --branch KB-XXXXX --limit 6
gh run watch <backend-run-id> --exit-status; gh run watch <frontend-run-id> --exit-status

# 3a. first time: stand up a burner
gravi burner start --build KB-XXXXX --ttl 8 --dataset default --sync
#    -> https://<id>.burner.gravitate.energy

# 3b. already have a burner, frontend/backend code change only:
gravi burner autosync trigger <id>            # in-place, data kept

# 3c. change was in deployment_main (index/migration/seed):
gravi burner recreate <id>                     # full redeploy + re-seed (or `start` a fresh one)
```
