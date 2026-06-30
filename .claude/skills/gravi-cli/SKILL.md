---
name: gravi-cli
description: >-
  Use the `gravi` CLI to manage and inspect Gravitate ("mom") infrastructure — authenticate
  to mom, list and inspect instances and burners, fetch an instance/burner's config + DB
  connection strings + credentials, query a burner's MongoDB directly with no tunnel, switch
  local dev between instance environments, mint access tokens (including role-override for E2E
  tests), manage the Atlas IP whitelist, share markdown pastes, and watch ArgoCD syncs. Use
  this whenever the user mentions gravi, mom, mom.gravitate.energy, a burner or instance's
  config / creds / connection string, pointing their local backend at an instance, getting a
  token for an instance or burner, querying burner data, or runs any `gravi ...` command — even
  if they don't name the skill. For spinning up or managing burner instances specifically (the
  `gravi burner` subcommand), use the gravi-burners skill instead; this skill covers everything else.
---

# gravi CLI

`gravi` is Gravitate's infrastructure management tool — a Click CLI that talks to **mom**
(`https://mom.gravitate.energy/api`), the platform that owns Gravitate **instances** (long-lived
environments like `dev`, `prod`, `caseys`, `wawa_test`) and **burners** (ephemeral test
environments, e.g. `burner01`, `tank`). It's installed as a `uv` tool at `~/.local/bin/gravi`.

## Authoritative reference

Confirm exact flags with `gravi <command> --help` (and `gravi <command> <subcommand> --help`).
The CLI is version-matched to the installed `gravi` and is the source of truth — this skill
captures the *workflows, gotchas, and orientation* that `--help` doesn't, so prefer reading the
help over guessing flags. Check the version with `gravi --version`.

## Auth model

Most commands need a valid mom session.

- `gravi login` — browser device-flow auth against mom. `gravi status` shows login state + token
  expiry; `gravi whoami` shows the current user + mom URL; `gravi logout` clears + revokes.
- `gravi token <instance_key>` mints access/refresh tokens for a specific instance/burner.
- For non-interactive/CI use, pass a Personal Access Token file globally: `gravi --token-file <path> ...`
  (overrides device-flow auth).
- `gravi tokens` manages CLI authorization tokens; `gravi instance` manages mom instances
  (admin-gated — only works if you have admin access).

If a command fails with an auth error, check `gravi status` first and re-`login` if expired.

## The credential model (important gotcha)

Mom **strips `username:password` out of every connection string** it returns, server-side — creds
never leave mom in URL form. The CLI **rehydrates** them client-side at use time, resolving from
this precedence:

1. environment variable
2. mom's credential store
3. `~/.config/gravitate/bb_tools.env`
4. 1Password

So if `gravi config` returns a conn string that won't connect, the creds aren't resolving. Manage
them with `gravi creds`:

- `gravi creds status` — show which DB creds are configured and where they resolve from
- `gravi creds check` — exit 0 if required creds are resolvable, 1 if missing (good for scripts)
- `gravi creds set` — set DB creds (written to `~/.config/gravitate/bb_tools.env`)
- `gravi creds prime` — print eval-able shell exports (`eval "$(gravi creds prime)"`)

`bbd-client` (`BBDClient.from_mom`) reads these same creds at construction time.

## Discover what you have access to

- `gravi instances` — list every instance + burner you can reach. `--json` for scripting,
  `--type burner|instance|all` to filter.
- `gravi config <key>` — get the full config (URLs, DB connection strings, etc.) for an instance
  key (`dev`, `prod`) or a burner id (`burner01`). `--format json|env` (env is handy for sourcing).
  Burners share the dev Atlas cluster; their dbs resolve to `burner_<id>_<service>` namespaces.

## Query a burner's MongoDB fast (no tunnel)

`gravi query <burner_id> <database> <collection>` runs a **read-only** query straight through the
mom API — no port-forward or tunnel needed. Great for quick data inspection.

**Example 1:** `gravi query tank backend orders --limit 5`
**Example 2:** `gravi query tank backend orders -f '{"status": "delivered"}' --count`
**Example 3:** `gravi query tank backend users --one -f '{"email": "admin@test.com"}'`
**Example 4:** `gravi query tank pricing price_v2 -s '{"effective_date": -1}' -l 20`

Flags: `-f/--filter`, `-p/--projection`, `-s/--sort` (all JSON), `-l/--limit`, `--count`, `--one`.

## Point local dev at an instance

`gravi switch <instance>` rewrites `backend/.env`, starts a Redis port-forward, and compares the
repo's current branch against the instance's expected branch. Run it from the
`supply_and_dispatch_aio` repo.

- `gravi switch caseys --checkout` — also checks out the instance's expected branch
- `gravi switch --no-pf wawa_test` — skip the port-forward
- `gravi switch --setup` — re-run interactive onboarding
- Subcommands: `status` (current instance, port-forward health, branch), `branch` (expected vs
  current, offer checkout), `pf start|...` (manage port-forwards), `sync` (force-refresh cached data)

## Tokens for E2E tests (role override)

`gravi token <instance>` can borrow a dataset-defined role without seeding a dedicated user — useful
in E2E tests. All burners allow role override.

- `gravi token tank --as-role "Inventory Costing"` — auth as that role instead of your default set
- `--add-role <ROLE>` adds a role on top of your existing ones; `--admin` requests an admin token
  (only honored if you have admin on the target); `--json` for scripting.

## Utilities

- `gravi paste` — share markdown: `upload` (file or stdin), `list`, `get` (to stdout), `open` (browser), `delete`.
- `gravi watch <app>` — subscribe to ArgoCD sync notifications via Slack DM. `--all` (every sync,
  persistent), `--cancel`, `--list`, `--json`. Default is one-shot (notify on next sync).
- `gravi mongo whitelist` — manage the MongoDB Atlas IP whitelist.

## Burner lifecycle → use the gravi-burners skill

For creating, starting, refreshing, extending, or deleting burners (the `gravi burner` subcommand)
and deploying a branch/PR to a burner, use the **gravi-burners** skill — it owns that workflow in
depth. This skill deliberately covers everything *except* burner lifecycle so the two compose
without overlap.

## Common workflows

- **"What's the connection string / creds for burner X?"** → `gravi config <X>` (then `gravi creds
  status` if the conn string won't connect).
- **"Check burner X's data without setting up a tunnel."** → `gravi query <X> <db> <collection> -f '...'`.
- **"Point my local backend at the caseys instance."** → `gravi switch caseys` (from `supply_and_dispatch_aio`).
- **"Get an admin token for burner X for an E2E test as role Y."** → `gravi token <X> --as-role "Y"` (add `--admin` if needed).
