# GraviFlow recruit and toolchain doctor

**Status:** design approved in conversation 2026-09-29; this spec is pending review.
**Repos:** `bestbuy_tools` (mom backend and frontend, gravi CLI) and `supply_and_dispatch_aio` (skills).

## Goal

A teammate presses **Recruit** on the GraviFlow page and gets a single command. They run it on their
laptop in an AIO checkout. The command first checks that their machine and access are set up for the
gate they picked. If they are, it starts a Claude session that works tickets at that gate, one subagent
per ticket.

This version covers **estimation and triage**. Build needs its own worktree, a plan gate and a
multi-turn SDLC, so it's out of scope here; it will get a separate `recruit build` later.

Success looks like this: someone with no setup runs the command and is told which gates they can work,
or exactly what's missing and how to fix it. Then Claude starts on tickets without any further setup.

## Decisions

| Decision | Choice | Why |
|---|---|---|
| Who's recruited | A teammate on their own laptop. No herdr, no fleet. | Bootstrapping is the goal |
| Shape | One lead Claude session that fans out one subagent per ticket | Keeps one agent per ticket (fresh context per ticket) with no worktrees; the lead paces calls to stay under mom's rate limit |
| Stop conditions | `--count`, `--minutes`, or an empty queue, whichever comes first; `--parallel` caps concurrency | Wanted as experiment knobs |
| Tool access | CLIs, not MCPs: `super` for Jira, and `gravi-axi` for Loki and Sentry through a mom proxy | One login, and access granted by role, not per-person tokens |
| Observability credentials | Held only by mom, as service credentials | Teammates never hold Sentry or Grafana tokens |
| Access limits | Per-user environment scoping and usage quotas on the proxy | Prod logs and events can contain customer data |

## Components

### 1. Mom observability proxy (backend)

These are read-only routes under `/observability/`.

**Caller authentication.** The caller's existing gravi/mom token, which must carry the new permission
**`observability:read`**. This permission is separate from `ticket_flow.agent`, so it can be granted
without flow access and revoked on its own.

**Environment scoping.** A grant of `observability:read` carries an **environment scope**:

- `nonprod`: the default when the permission is granted. It covers every dev and test instance key.
- `prod:<key>` entries, or `prod:*`: explicit grants, one environment at a time or all of them. Only a
  mom admin can make these grants.

A request for an environment outside the caller's scope is refused with 403, and the response names the
missing grant.

**Usage limits** apply to each user and are configurable by an admin:

- **Rate:** requests per minute; the default is 30.
- **Daily quota:** queries per day; the default is 500. Log lines returned per day; the default is
  50,000.
- A request over a limit gets a 429 carrying `Retry-After` and the limit that was hit. The CLI shows
  both.

**Loki routes:**

- `POST /observability/logs/query`, with `{env, logql_selector_filters, since | start/end, limit, direction}`.
  - Mom maps `env` to the Loki namespace using the instance-key crosswalk from the triage references,
    so the caller never names a namespace. Mom injects `kubernetes_namespace_name`, and it refuses a
    selector that names a different namespace.
  - The window is capped at 4 days, with a line cap on each request.
  - Mom calls Loki through Grafana's datasource proxy, using a Grafana **service account** (Viewer).
- `POST /observability/logs/count` returns a metric count (`count_over_time`) under the same limits.

**Sentry routes:**

- `GET /observability/sentry/issues/{short_id}` returns the issue's details, event count, and first
  and last seen times, plus the latest event with its stack trace.
- `GET /observability/sentry/issues?env=&query=&since=` searches issues.
- The prod environment uses hosted Sentry (`capspire`); test and dev use self-hosted `sentry-dev`. Mom
  picks the instance from `env`.
- Each instance has its own **internal integration** token with `event:read`, `project:read` and
  `org:read`. The proxy offers no write, resolve or assign actions.

**Credential storage.** The Grafana service-account token and the two Sentry integration tokens live
in Google Secret Manager. Kuberist's `sync-secrets.py` syncs them into mom's environment. Rotating a
token means changing it in Secret Manager. Creating them is a one-time admin setup.

**Audit.** Every call is logged with the caller, the env, the route, the query or issue, the rows
returned and the time taken.

### 2. `gravi-axi logs` and `gravi-axi sentry` (CLI)

These are thin clients over the proxy. They print TOON output, and `--json` gives the raw response.

```
gravi-axi logs <env> '<LogQL filter>' [--container backend-web] [--since 24h] [--limit 200]
gravi-axi logs count <env> '<LogQL filter>' [--since 24h]
gravi-axi sentry issue BEST_BUY_SERVICES-1GXC
gravi-axi sentry search <env> '<query>' [--since 14d]
```

A 403 prints the grant that's missing. A 429 prints the limit and the retry time.

### 3. `gravi-axi flow doctor [estimate|triage]` (CLI)

Doctor runs each gate's checks. It prints ✅, ⚠️ or ❌ for each, together with the exact fix, and exits
non-zero when the requested gate has a ❌.

| Check | Estimate | Triage | How it's checked | Fix it prints |
|---|---|---|---|---|
| gravi-cli version ≥ required, not shadowed | ❌ | ❌ | `which -a gravi-axi`, `--version` | `uv tool install --force gravi-cli==X`, and name the copy that's shadowing it |
| Logged in to mom, with `ticket_flow.agent` | ❌ | ❌ | read-only `flow ls --limit 1` | `gravi login`, then ask for the Ticket Flow agent role |
| Jira through `super` | ❌ | ❌ | `super read jira://<a known key>/status` | the `super` install steps |
| Claude Code | ❌ | ❌ | `claude --version` | the install link |
| AIO checkout, with the skills present | ❌ | ❌ | the git remote matches; `.claude/skills/snd-flow-bugs-{estimate,triage}` and `snd-flow-recruit` exist | clone or `--repo`; how to get the skills |
| Git access to RC | | ❌ | `git ls-remote <remote> RC` | the GitHub access steps |
| `observability:read`, nonprod | | ❌ | a small `logs count` call on a nonprod env | ask for the Observability Reader role |
| `observability:read`, prod | | ⚠️ | report the prod env keys in the caller's scope | ask an admin for a prod grant; triage runs, but prod evidence is limited |
| Prod data, read-only | | ⚠️ | `gravi query <a test env> backend order_v2 --count` | the gravi query access steps |
| Burners | | ⚠️ | `gravi burner list` | the burner access steps; triage runs without live repros |

A ⚠️ never blocks a gate. The triage skill already reports evidence it couldn't check, so a limited run
still produces an honest report.

`gravi-axi flow doctor` with no gate prints one table covering all gates.

### 4. `gravi-axi flow recruit <estimate|triage>` (CLI)

```
gravi-axi flow recruit triage [--parallel 3] [--count 10] [--minutes 60] [--repo PATH]
```

1. Runs doctor for the gate. It stops on any ❌, after printing the fixes.
2. Resolves the AIO checkout: `--repo`, or the current directory. It fetches RC, read-only.
3. Launches `claude` in the checkout with a prompt that invokes the **`snd-flow-recruit`** skill,
   passing the gate and the three limits.

The defaults are `--parallel 3`, `--count 5` and `--minutes 60`. A first run stays cheap by default.

### 5. `snd-flow-recruit` lead skill (AIO)

The lead loop:

1. Keep up to `--parallel` subagents running. For each free slot, run
   `gravi-axi flow claim <gate> --name snd-flow-recruit --run-ref <lead id>`. That gets the next ticket
   in work order and starts its run.
2. Start a subagent for that ticket with that gate's skill (`snd-flow-bugs-estimate` or
   `snd-flow-bugs-triage`). Its instruction is the skill plus the key. The skill already continues a run
   the caller holds (`--run current`).
3. As each subagent finishes, report one line: key, outcome, ratings or recommendation. Then fill the
   slot again.
4. Stop starting new tickets when `--count` tickets have started, when `--minutes` has passed, or when
   `claim` returns nothing. Let the running subagents finish, then print a summary.

Rules for the lead:

- The lead never reads ticket content or reports. It passes keys only, so no context crosses from one
  ticket to another.
- Rate limits: exit 5 means wait and retry, as the gate skills already do. The lead also spreads
  claims out, starting at most one every few seconds.
- A subagent that fails its run is counted and reported. The lead doesn't retry it; the ticket goes back
  in the queue.

**The claim hand-off.** The lead's `claim` starts a run under the lead's principal. The gate skills
begin with `flow start`, so they need to accept a run that's already open. **Required change:** the
estimate and triage skills skip `flow start` when told "your run is already started, run id X", and
pass `--run X` on their writes.

### 6. Recruit button (frontend)

This is a **Recruit** button in the GraviFlow page's top bar. It opens a dialog with:

- A gate picker (Estimate, Triage), with a count of tickets currently waiting at each gate.
- Inputs for parallel, count and minutes, set to the defaults above.
- The generated command, with a copy button.
- "First time? Run `gravi-axi flow doctor triage` first", with the one-line install for `gravi-cli`.

The dialog only displays text; nothing runs from the browser.

### 7. Skill updates (AIO)

- **Triage:** the Sentry and Loki guidance in `sentry-and-logs.md`, and the environment crosswalk in
  `environments.md`, move to `gravi-axi sentry` and `gravi-axi logs <env>`. The mapping now happens on
  the server, so the crosswalk table shrinks to "pass the instance key". Both references drop MCP
  instructions, and the Atlassian MCP fallback for Jira goes too, since doctor requires `super`.
- **Estimate and triage:** both accept an already-started run (see the claim hand-off in component 5).

## Error handling

- **Doctor** checks don't depend on each other: one failed check never hides the others.
- **Recruit** exits non-zero with doctor's output when the gate is blocked.
- **The proxy** returns 403 (scope, with the missing grant named), 429 (limit, `Retry-After`), 422
  (a disallowed LogQL selector or window) or 502 (the upstream is down, with the upstream named). The
  CLI prints each in one line.
- **The lead** survives a subagent failing, and reports it.

## Testing

- **Proxy:** unit tests for env→namespace/instance mapping, scope checks (nonprod, a single prod key,
  prod wildcard, refused), selector injection and rejection, window and line caps, rate and quota
  limits, and audit writes. Upstream calls are mocked.
- **CLI:** tests for each command's output and each error mapping; doctor checks run against faked
  command results.
- **Frontend:** tests for the dialog's command generation and the waiting counts.
- **End to end,** by hand: a fresh machine (or a clean shell user) runs doctor, fixes what it reports,
  then runs `recruit estimate --parallel 2 --count 2` against the real queue.

## Out of scope

- `recruit build`: a worktree, a plan gate, and a long-running session per ticket.
- Metrics and Prometheus through the proxy.
- Retiring the Grafana and Sentry MCPs for people who use them outside the flow.

## Open items

- The admin who creates the Grafana service account and the two Sentry integration tokens.
- Which roles include `observability:read` by default, and who can grant prod scope.
