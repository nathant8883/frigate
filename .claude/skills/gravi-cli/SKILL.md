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
  if they don't name the skill. Also covers `gravi-axi`, the agent-facing sister command: reading
  Loki logs and Sentry issues for an environment through mom (`gravi-axi logs`, `gravi-axi sentry`),
  the GraviFlow ticket commands (`gravi-axi flow ls/get/claim/estimate/report/doctor/recruit`),
  and its exit codes. For spinning up or managing burner instances specifically (the `gravi burner`
  subcommand), use the gravi-burners skill instead; this skill covers everything else.
---

# gravi CLI

`gravi` is Gravitate's infrastructure management tool — a Click CLI that talks to **mom**
(`https://mom.gravitate.energy/api`), the platform that owns Gravitate **instances** (long-lived
environments like `dev`, `prod`, `caseys`, `wawa_test`) and **burners** (ephemeral test
environments, e.g. `burner01`, `tank`). It's installed as a `uv` tool at `~/.local/bin/gravi`, together with `gravi-axi` (below).

**Check which copy you're running.** An older `gravi-cli` installed with pip into a pyenv Python can
sit ahead of the uv install on `PATH`, and then new subcommands appear missing ("No such command
'flow'"). `which -a gravi-axi` lists every copy, and `gravi-axi flow doctor` flags a shadowed one.
Upgrade both copies: `uv tool install --force gravi-cli==X`, and for pyenv
`uv pip install --python ~/.pyenv/versions/<v>/bin/python --upgrade gravi-cli==X`.

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

The database argument is the logical name (`backend`, `payroll`, `price`…), never `<key>_backend`.
It also works on instance keys, for example `gravi query jhoil backend order_v2 --count`. Limits: 10
documents by default, `-l` capped at 1000 (a capped result carries `"truncated": true`), and `$oid`
filters are refused, so filter on another field. `gravi query sup …` fails because of a DNS issue on
mom's side; use BBDClient `from_config("sup")` for Sup.

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

## gravi-axi: the agent-facing command

`gravi-axi` ships in the same package, uses the same login, and talks to the same mom, but its output is
shaped for agents:

- **Output is TOON by default** (compact rows, `--json` to escape it). `#` lines are summaries and
  `# next:` hints.
- **Errors go to stderr as TOON, with stable exit codes.** It never prompts.
- **The full command list is in `mom/cli/AXI.md`** in bestbuy_tools, or run
  `gravi-axi <group> --help`.

| Exit | Meaning | What to do |
|---|---|---|
| 0 | success | |
| 2 | API or generic error (`status:` gives the HTTP code) | read the message |
| 3 | not authenticated | `gravi login` |
| 4 | permission denied | the message names the missing role or grant; ask a mom admin |
| 5 | rate limited | wait 60s and retry the same command |
| 6 | burner capacity reached | |
| 7 | conflict: another agent holds the ticket, or the gate isn't next | stop, don't retry |

### Logs and Sentry through mom

Mom holds the Grafana and Sentry credentials, so you don't need your own. Pass an **instance key**;
mom maps it to the Loki namespace and Sentry environment and checks your access.

```bash
gravi-axi logs jhoil '| level="ERROR" |= "KeyError"' --container backend-web --since 24h
gravi-axi logs count jhoil '| level=~"ERROR|CRITICAL"' --container backend-actors --since 4d
gravi-axi sentry issues jhoil                                   # unresolved, most frequent first
gravi-axi sentry issue BEST_BUY_SERVICES-1GJT --env jhoil --markdown   # stack, snippets, locals
gravi-axi sentry search jhoil 'KeyError' --since 30d
```

- **Access:** the Observability Reader role covers non-prod environments. Prod needs
  `observability:prod:<key>` or `observability:prod:*` from an admin, and a 403 names the missing grant.
- **Log filters are LogQL pipeline stages only** (each starting with `|`). A query spans at most
  4 days and returns at most 1000 lines, and a count over more than an hour needs `--container` or a
  filter.
- **Don't confuse the two `logs` forms.** `gravi-axi logs <burner> <pod>` reads a **burner pod's**
  logs; `gravi-axi logs <env> '<filters>'` queries **Loki** for an environment.

### GraviFlow tickets

`gravi-axi flow` drives the agent ticket flow (the BugFlow page in mom). The skills
`snd-flow-bugs-estimate`, `snd-flow-bugs-triage`, `snd-flow-bugs-build` and `snd-flow-recruit` say how
to use each gate, so follow them; this is the map:

```bash
gravi-axi flow ls --next triage                 # tickets an agent can start at a gate, in work order
gravi-axi flow get KB-12345 --full              # one ticket's record: ratings, reports, guidance, links
gravi-axi flow claim triage                     # start a run on the next free ticket
gravi-axi flow doctor triage                    # is this machine set up for the gate? exit 2 if blocked
gravi-axi flow recruit triage --parallel 3 --count 5 --minutes 60   # doctor, then launch a Claude lead
```

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
- **"What's failing on jhoil?"** → `gravi-axi sentry issues jhoil`, then `gravi-axi sentry issue <id> --env jhoil --markdown`.
- **"Is this error still happening?"** → `gravi-axi logs count <key> '| level="ERROR" |= "<text>"' --container backend-web --since 4d`.
- **"Can I work triage from this laptop?"** → `gravi-axi flow doctor triage`.
