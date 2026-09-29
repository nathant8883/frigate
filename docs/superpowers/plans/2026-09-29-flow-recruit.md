# GraviFlow Recruit and Toolchain Doctor: Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A teammate presses Recruit on the GraviFlow page, copies one command, and on their laptop that command checks their toolchain and starts a Claude lead. The lead works estimation or triage tickets, one subagent per ticket.

**Architecture:**
- Mom gains a read-only observability proxy (Loki through Grafana, plus Sentry) that holds the service credentials itself and scopes every call to the environments the caller has been granted.
- gravi-cli gains `logs`, `sentry`, `flow doctor` and `flow recruit`.
- The AIO repo gains a lead skill, `snd-flow-recruit`. The two existing gate skills stop using MCPs and start using the new CLI commands.

**Tech Stack:**
- Mom backend: FastAPI with Beanie, httpx, and pytest with `httpx.MockTransport`.
- gravi-cli: click and requests, with pytest, `CliRunner` and `responses`.
- Mom frontend: React/TS with antd and vitest.
- AIO skills: markdown.

**Spec:** `docs/superpowers/specs/2026-09-29-flow-recruit-design.md` (frigate). Read it before starting.

## Global Constraints

- **Repos:** `bestbuy_tools` for mom and gravi-cli, which is rebase-only. Work on a branch, squash it, rebase onto `bb_tools/mom` and push. There's no PR, and pushing needs no approval. `supply_and_dispatch_aio` for the skills, which are local skills under `.claude/skills/` and aren't committed.
- The proxy is **read-only**: no Sentry write, resolve or assign, and no Loki push.
- Callers name an **instance key**, never a namespace or Sentry org. Mom does the mapping.
- **Environment scope:** a grant of `observability:read` gives access to non-prod environments only. Prod needs `observability:prod:<key>` or `observability:prod:*`, granted by an admin.
- **No usage limits** on the proxy, meaning no per-user rate limit and no quota. Per-request caps stay: a window of at most **4 days** per Loki query, at most **1000 lines** per response, and exactly one namespace.
- Mom's credentials live only in mom settings, synced from Google Secret Manager by kuberist `sync-secrets.py`. Teammates never hold Grafana or Sentry tokens.
- **Every proxy call is audit-logged:** caller, env, route, query, rows returned and duration.
- `recruit` defaults: `--parallel 3`, `--count 5`, `--minutes 60`. The run stops at whichever limit comes first, or when the queue is empty.
- **Scope of this plan:** estimation and triage only. `recruit build` is out.
- Mom code style: almost no comments, one line maximum each, and docstrings of two lines maximum. Match the existing patterns cited in each task.

## Review Focus

1. **A LogQL filter that tries to escape its namespace** must be refused with 422, not run against another tenant. Examples are a filter containing `{...}`, one naming `kubernetes_namespace_name`, and one using `or` or `unless` across selectors. Tested in Task 5.
2. **An unknown env key** returns 404 with a hint to run `gravi instances`. A prod key the caller wasn't granted returns 403 naming the exact missing grant. Neither may return 500. Tested in Tasks 3 and 4.
3. **Environment config not mounted** (local dev, where `/app/environment/config.json` is missing) returns a 503 saying so, rather than a false "unknown env" 404. Tested in Task 3.
4. **Doctor run outside an AIO checkout, or with dangling skill symlinks** reports ❌ with the fix. A missing tool is a ❌ result, never a Python traceback. Tested in Task 10.
5. **Recruit when the gate's queue is empty** prints "nothing waiting at <gate>" and exits 0 without launching Claude. **Recruit when `claude` isn't installed** is caught by doctor. Tested in Task 11.

---

## Part A: mom backend (crew: Bravo, `bestbuy_tools`)

### Task 1: Settings and upstream clients (Grafana/Loki, Sentry)

**Files:**
- Modify: `mom/backend/settings.py`, next to the Jira block at lines ~192-195
- Create: `mom/backend/observability/__init__.py`, `mom/backend/observability/clients.py`
- Test: `mom/backend/tests/test_observability_clients.py`

**Interfaces:**
- Produces:
  - `LokiClient(base_url, token, transport=None)` with `async query_range(logql: str, start: datetime, end: datetime, limit: int, direction: str) -> dict`
  - `SentryClient(base_url, org, token, transport=None)` with `async issue(short_id: str) -> dict`, `async latest_event(issue_id: str) -> dict`, and `async search(project: str, environment: str, query: str, stats_period: str) -> list[dict]`
  - `get_loki_client() -> LokiClient | None`
  - `get_sentry_client(kind: Literal["hosted","self_hosted"]) -> SentryClient | None`
  - `class UpstreamError(RuntimeError)` with `.service` and `.status`

- [ ] **Step 1: Add the settings.** Use the same pattern as Jira. They're empty by default, and `MOM_` is added as a prefix automatically.

```python
    # Observability proxy
    grafana_url: str = ""
    grafana_loki_datasource_uid: str = "loki"
    grafana_service_token: str = ""
    sentry_hosted_url: str = "https://us.sentry.io"
    sentry_hosted_org: str = "capspire"
    sentry_hosted_token: str = ""
    sentry_self_hosted_url: str = "https://sentry-dev.gravitate.energy"
    sentry_self_hosted_org: str = "sentry"
    sentry_self_hosted_token: str = ""
```

- [ ] **Step 2: Write failing tests** that use `httpx.MockTransport`, following the style of `tests/test_release_manager_argocd_client.py`.

```python
import httpx, pytest
from datetime import datetime, timedelta, timezone
from observability.clients import LokiClient, SentryClient, UpstreamError

async def test_loki_query_range_hits_grafana_datasource_proxy():
    seen = {}
    def handler(req):
        seen["url"], seen["auth"] = str(req.url), req.headers["Authorization"]
        return httpx.Response(200, json={"status": "success", "data": {"resultType": "streams", "result": []}})
    c = LokiClient("https://grafana.test", "tok", datasource_uid="loki", transport=httpx.MockTransport(handler))
    end = datetime(2026, 9, 29, tzinfo=timezone.utc)
    await c.query_range('{kubernetes_namespace_name="jhoil"}', end - timedelta(hours=1), end, 100, "backward")
    assert seen["url"].startswith("https://grafana.test/api/datasources/proxy/uid/loki/loki/api/v1/query_range?")
    assert seen["auth"] == "Bearer tok"

async def test_loki_upstream_502_raises_upstream_error():
    c = LokiClient("https://grafana.test", "tok", transport=httpx.MockTransport(lambda r: httpx.Response(502, text="bad gateway")))
    now = datetime.now(timezone.utc)
    with pytest.raises(UpstreamError) as e:
        await c.query_range("{}", now, now, 1, "backward")
    assert e.value.service == "loki" and e.value.status == 502

async def test_sentry_issue_and_latest_event():
    def handler(req):
        if req.url.path.endswith("/issues/"):
            return httpx.Response(200, json=[{"id": "42", "shortId": "BEST_BUY_SERVICES-1GXC", "count": "48"}])
        return httpx.Response(200, json={"eventID": "e1", "entries": []})
    c = SentryClient("https://us.sentry.io", "capspire", "tok", transport=httpx.MockTransport(handler))
    issue = await c.issue("BEST_BUY_SERVICES-1GXC")
    assert issue["id"] == "42"
    assert (await c.latest_event("42"))["eventID"] == "e1"
```

- [ ] **Step 3:** Run `cd mom/backend && uv run pytest tests/test_observability_clients.py -v`. Expected: FAIL with `ModuleNotFoundError: observability`.
- [ ] **Step 4: Implement `observability/clients.py`.** Open an `httpx.AsyncClient` per call, as `jira_release/client.py` does, with `transport=self._transport` when one is set. Any non-2xx or `httpx.HTTPError` raises `UpstreamError(service, status, text)`.
  - **Loki:** `GET {grafana_url}/api/datasources/proxy/uid/{uid}/loki/api/v1/query_range` with params `query`, `start` and `end` (nanoseconds as a string), `limit` and `direction`.
  - **Sentry issue:** `GET {base}/api/0/organizations/{org}/issues/?query={short_id}&shortIdLookup=1`, returning the first element. Raise `UpstreamError("sentry", 404, ...)` when the list is empty.
  - **Sentry latest event:** `GET {base}/api/0/organizations/{org}/issues/{id}/events/latest/`.
  - **Sentry search:** `GET {base}/api/0/organizations/{org}/issues/?project={project}&environment={env}&query={q}&statsPeriod={p}`.
  - The factories return `None` when their token is empty.
- [ ] **Step 5:** Run the tests. Expected: PASS.
- [ ] **Step 6:** Commit with `feat(observability): Grafana/Loki and Sentry upstream clients`.

### Task 2: Permissions, the default role and the scope check

**Files:**
- Modify: `mom/backend/rbac/permissions.py`, adding to `class Permission` next to line 40
- Modify: `mom/backend/rbac/default_roles.py`, adding to `DEFAULT_ROLES`
- Create: `mom/backend/observability/access.py`
- Test: `mom/backend/tests/test_observability_access.py`

**Interfaces:**
- Consumes: `get_user_permissions(user) -> set[str]` from `rbac/auth.py`
- Produces:
  - `Permission.OBSERVABILITY_READ = "observability:read"`
  - the constant `PROD_GRANT_PREFIX = "observability:prod:"`
  - `async def allowed_env(user, env_key: str, env_type: str) -> tuple[bool, str | None]`, which returns `(ok, missing_grant)`

- [ ] **Step 1: Write the failing tests.**

```python
import pytest
from observability.access import allowed_env

@pytest.mark.parametrize("perms,key,typ,ok,missing", [
    ({"observability:read"}, "sup", "dev", True, None),
    ({"observability:read"}, "loves_test", "test", True, None),
    ({"observability:read"}, "jhoil", "prod", False, "observability:prod:jhoil"),
    ({"observability:read", "observability:prod:jhoil"}, "jhoil", "prod", True, None),
    ({"observability:read", "observability:prod:*"}, "bboil", "prod", True, None),
    ({"observability:prod:*"}, "bboil", "prod", False, "observability:read"),
    ({"*"}, "bboil", "prod", True, None),
    (set(), "sup", "dev", False, "observability:read"),
])
async def test_allowed_env(mocker, perms, key, typ, ok, missing):
    mocker.patch("observability.access.get_user_permissions", return_value=perms)
    assert await allowed_env(object(), key, typ) == (ok, missing)
```

- [ ] **Step 2:** Run `uv run pytest tests/test_observability_access.py -v`. Expected: FAIL with an import error.
- [ ] **Step 3: Implement it.**

```python
from rbac.auth import get_user_permissions

PROD_GRANT_PREFIX = "observability:prod:"

async def allowed_env(user, env_key: str, env_type: str) -> tuple[bool, str | None]:
    perms = await get_user_permissions(user)
    if "*" in perms:
        return True, None
    if "observability:read" not in perms:
        return False, "observability:read"
    if env_type != "prod":
        return True, None
    if f"{PROD_GRANT_PREFIX}*" in perms or f"{PROD_GRANT_PREFIX}{env_key}" in perms:
        return True, None
    return False, f"{PROD_GRANT_PREFIX}{env_key}"
```

Add `OBSERVABILITY_READ = "observability:read"` to `Permission`. Add the default role to `DEFAULT_ROLES`:

```python
    {"name": "observability_reader", "display_name": "Observability Reader",
     "description": "Read Loki logs and Sentry issues for non-prod environments through mom",
     "permissions": [Permission.OBSERVABILITY_READ.value], "is_system_role": True,
     "instance_access_rules": [], "specific_instance_access": []},
```

Prod grants are the raw permission strings `observability:prod:<key>` and `observability:prod:*`, added by an admin to a custom role. They are deliberately left out of the `Permission` enum, because they're keyed by environment. If the role editor validates permissions against the enum, allow the `observability:prod:` prefix there too.

- [ ] **Step 4:** Run the tests. Expected: PASS.
- [ ] **Step 5:** Commit with `feat(observability): observability:read permission, reader role, env scope check`.

### Task 3: The env resolver

**Files:**
- Create: `mom/backend/observability/envs.py`
- Test: `mom/backend/tests/test_observability_envs.py`

**Interfaces:**
- Consumes: `get_all_instance_configs() -> dict` from `instance/db_utils.py`
- Produces:
  - `@dataclass class ResolvedEnv: key: str; type: str; namespace: str; sentry_kind: Literal["hosted","self_hosted"]; sentry_environment: str; sentry_project: str`
  - `def resolve_env(key: str) -> ResolvedEnv`, which raises `EnvConfigMissing` (so the route returns 503) or `UnknownEnv` (404)

- [ ] **Step 1: Find the real Sentry environment tags.** Before writing the rules, dump them from the deployments. In kuberist, run `git grep -n "ENVIRONMENT:" origin/master -- 'clusters/*/configmap.yaml'`. For environments not yet migrated, grep `deployment_configs/*/env-cm.yaml`. Record every key whose tag breaks the rules below in `SENTRY_ENV_OVERRIDES`. Known exceptions so far: `coleman`, `holiday`, `hucks` and `rbs` have no `-prod` suffix; `kt_test` maps to `kttest`; `7_eleven` maps to `7_eleven-prod`; Costco keys use underscores (`costco_us`, `costco_test_cn`).
- [ ] **Step 2: Write the failing tests.**

```python
import pytest
from observability import envs

CFG = {"jhoil": {"type": "prod"}, "sup": {"type": "dev"}, "loves_test": {"type": "test"},
       "caseys": {"type": "prod"}, "costco_us": {"type": "prod"}, "costco_test_cn": {"type": "test"}}

@pytest.fixture(autouse=True)
def cfg(mocker):
    mocker.patch("observability.envs.get_all_instance_configs", return_value=CFG)

def test_prod_key_maps_to_hosted_sentry_and_dash_namespace():
    r = envs.resolve_env("jhoil")
    assert (r.namespace, r.sentry_kind, r.sentry_environment) == ("jhoil", "hosted", "jhoil-prod")

def test_test_key_maps_to_self_hosted():
    r = envs.resolve_env("loves_test")
    assert (r.namespace, r.sentry_kind, r.sentry_environment) == ("loves-test", "self_hosted", "loves-test")

def test_overrides():
    assert envs.resolve_env("caseys").namespace == "caseys-prod"
    assert envs.resolve_env("costco_us").sentry_environment == "costco_us"
    assert envs.resolve_env("costco_test_cn").namespace == "costco-test-cn"

def test_unknown_key():
    with pytest.raises(envs.UnknownEnv):
        envs.resolve_env("nope")

def test_config_not_mounted(mocker):
    mocker.patch("observability.envs.get_all_instance_configs", return_value={})
    with pytest.raises(envs.EnvConfigMissing):
        envs.resolve_env("jhoil")
```

- [ ] **Step 3:** Run `uv run pytest tests/test_observability_envs.py -v`. Expected: FAIL.
- [ ] **Step 4: Implement it.**
  - Namespace: `NAMESPACE_OVERRIDES.get(key, key.replace("_", "-"))`, with `caseys` mapping to `caseys-prod`.
  - Sentry kind: `hosted` when `type == "prod"`, `self_hosted` otherwise.
  - Sentry environment: `SENTRY_ENV_OVERRIDES` if the key is listed. Otherwise, keys starting `costco` use the key itself. Otherwise a prod key gets `f"{key}-prod"`, and any other key gets `key.replace("_", "-")`.
  - Sentry project: `"best_buy_services"` for both instances.
  - Errors: raise `EnvConfigMissing` when the config dict is empty, and `UnknownEnv(key)` when the key is missing.
- [ ] **Step 5:** Run the tests. Expected: PASS.
- [ ] **Step 6:** Commit with `feat(observability): instance-key to namespace/sentry resolver`.

### Task 4: The audit model and the caller dependency

**Files:**
- Create: `mom/backend/observability/model.py` (`ObservabilityAudit`) and `mom/backend/observability/deps.py`
- Modify: `mom/backend/tests/conftest.py`, adding `ObservabilityAudit` to the `document_models` list in `db_client` at line ~220
- Test: `mom/backend/tests/test_observability_deps.py`

**Interfaces:**
- Consumes: `resolve_env` (Task 3), `allowed_env` (Task 2), `current_user` (used by `rbac/auth.require_permission`)
- Produces:
  - `class ObservabilityAudit(Document)` with the fields `at, caller, env, route, query, rows, duration_ms, status`, stored in the collection `observability_audit`
  - `async def scoped_env(env: str, user=Depends(current_user)) -> tuple[User, ResolvedEnv]`, which raises 503, 404 or 403 with the detail strings below
  - `async def audit(caller, env, route, query, rows, duration_ms, status)`

- [ ] **Step 1: Write the failing tests.** Use the `db_client` and `auth_header_for` fixtures, plus a throwaway router that uses `scoped_env`, mounted on a test app.

```python
async def test_scoped_env_errors(http, auth_header_for, reader_user, mocker):
    mocker.patch("observability.envs.get_all_instance_configs", return_value={"jhoil": {"type": "prod"}, "sup": {"type": "dev"}})
    h = auth_header_for(reader_user)
    assert (await http.get("/_t/nope", headers=h)).status_code == 404
    r = await http.get("/_t/jhoil", headers=h)
    assert r.status_code == 403 and "observability:prod:jhoil" in r.text
    assert (await http.get("/_t/sup", headers=h)).status_code == 200

async def test_scoped_env_503_when_config_missing(http, auth_header_for, reader_user, mocker):
    mocker.patch("observability.envs.get_all_instance_configs", return_value={})
    r = await http.get("/_t/sup", headers=auth_header_for(reader_user))
    assert r.status_code == 503 and "environment config" in r.text
```

The `reader_user` fixture follows the `reviewer_headers` pattern in `test_agent_ticket_flow_endpoints.py`: insert the `observability_reader` role from `DEFAULT_ROLES`, call `invalidate_role_cache()`, and assign the role to the user.

- [ ] **Step 2:** Run the tests. Expected: FAIL.
- [ ] **Step 3: Implement `deps.py`.** The detail strings:
  - 404: `f"Unknown environment '{env}'. Run: gravi instances"`
  - 403: `f"Requires {missing} for {env}. Ask a mom admin."`
  - 503: `"The environment config isn't mounted on this mom. Observability is unavailable."`
- [ ] **Step 4:** Run the tests. Expected: PASS.
- [ ] **Step 5:** Commit with `feat(observability): env-scoped caller dependency and audit model`.

### Task 5: The Loki routes

**Files:**
- Create: `mom/backend/observability/logql.py` (the selector guard) and `mom/backend/observability/endpoints.py`
- Modify: `mom/backend/app.py`. Import `from observability.endpoints import routers as observability_routers` and run `all_routers.extend(observability_routers)`, following the ticket-flow pattern at lines 65 and 263.
- Test: `mom/backend/tests/test_observability_logql.py` and `mom/backend/tests/test_observability_endpoints.py`

**Interfaces:**
- Consumes: `LokiClient` (Task 1), `scoped_env` and `audit` (Task 4)
- Produces:
  - `def build_query(namespace: str, container: str | None, filters: str) -> str`, which raises `LogqlRejected`
  - `POST /observability/logs/query` with body `{env, filters, container?, since?: "24h", start?, end?, limit?: 200, direction?: "backward"}`, returning `{env, namespace, lines: [{ts, container, level, message}], truncated: bool}`
  - `POST /observability/logs/count` with body `{env, filters, container?, since?}`, returning `{env, count, by_container: {...}}`

- [ ] **Step 1: Write the failing guard tests.** These cover Review Focus 1.

```python
import pytest
from observability.logql import build_query, LogqlRejected

def test_builds_selector_with_injected_namespace():
    q = build_query("jhoil", "backend-web", '| level="ERROR" |= "KeyError"')
    assert q == '{kubernetes_namespace_name="jhoil", kubernetes_container_name="backend-web"} | level="ERROR" |= "KeyError"'

@pytest.mark.parametrize("bad", [
    '{kubernetes_namespace_name="bboil"}',
    '| json } or {kubernetes_namespace_name="bboil"',
    'unless {app="x"}',
    'kubernetes_namespace_name="bboil"',
    '|= "x" or on() vector(1)',
])
def test_rejects_selector_escapes(bad):
    with pytest.raises(LogqlRejected):
        build_query("jhoil", None, bad)

def test_filters_must_start_with_pipe_or_be_empty():
    assert build_query("sup", None, "") == '{kubernetes_namespace_name="sup"}'
    with pytest.raises(LogqlRejected):
        build_query("sup", None, 'rate({x="y"}[5m])')
```

- [ ] **Step 2: Implement `logql.py`.** Reject filters containing `{`, `}`, the words `or`, `unless`, `and`, `on(` or `ignoring(` outside quotes, or the text `kubernetes_namespace_name`. Non-empty filters must start with `|`. The container must match `^[a-z0-9-]+$`.
- [ ] **Step 3: Write the failing endpoint tests.** Mock the Loki client with `MockTransport` through a patched `get_loki_client`.

```python
async def test_logs_query_caps_window_and_limit(http, reader_headers, loki_mock):
    r = await http.post("/observability/logs/query", headers=reader_headers,
                        json={"env": "sup", "filters": '| level="ERROR"', "since": "10d", "limit": 5000})
    assert r.status_code == 422 and "4 days" in r.text
    r = await http.post("/observability/logs/query", headers=reader_headers,
                        json={"env": "sup", "filters": '| level="ERROR"', "since": "24h", "limit": 5000})
    assert r.status_code == 200
    assert loki_mock.last_params["limit"] == "1000"

async def test_logs_query_rejects_escape_with_422_and_audits(http, reader_headers, loki_mock):
    r = await http.post("/observability/logs/query", headers=reader_headers,
                        json={"env": "sup", "filters": '{kubernetes_namespace_name="bboil"}'})
    assert r.status_code == 422
    assert loki_mock.calls == 0
    assert await ObservabilityAudit.find(ObservabilityAudit.status == 422).count() == 1

async def test_logs_upstream_502_maps_to_502_naming_loki(http, reader_headers, loki_502): ...
```

- [ ] **Step 4: Implement the routes.**
  - `router = APIRouter(prefix="/observability", dependencies=[Depends(valid_access_token)])`
  - Parse `since` with the regex `^(\d+)([mhd])$`. A window over 4 days gets a 422 reading "Queries are capped at 4 days; walk back in steps".
  - Clamp `limit` to 1000. `truncated` is true when the number of lines equals the limit.
  - `count` runs `sum by (kubernetes_container_name) (count_over_time(<query> [<since>]))` through `query_range` with `step=since`, which the Loki client takes as an optional param. It sums the last values.
  - Every path, success or error, calls `audit(...)`.
  - The route catches `UpstreamError` and returns 502 with `{"detail": f"{e.service} returned {e.status}"}`.
- [ ] **Step 5:** Run `uv run pytest tests/test_observability_logql.py tests/test_observability_endpoints.py -v`. Expected: PASS.
- [ ] **Step 6:** Commit with `feat(observability): env-scoped Loki query and count routes`.

### Task 6: The Sentry routes

**Files:**
- Modify: `mom/backend/observability/endpoints.py`
- Test: `mom/backend/tests/test_observability_endpoints.py`

**Interfaces:**
- Produces:
  - `GET /observability/sentry/issues/{short_id}?env=<key>`, returning `{short_id, title, culprit, count, first_seen, last_seen, environments, url, latest_event: {id, date, message, frames: [{file, function, line, context}]}}`
  - `GET /observability/sentry/issues?env=&query=&since=14d`, returning `{issues: [{short_id, title, count, last_seen, url}]}`

- [ ] **Step 1: Write the failing tests.**
  - A prod env issue uses the hosted client, and a test env uses the self-hosted client. Assert which mock was called.
  - A search sends `environment=<resolved sentry_environment>` and `project=best_buy_services`.
  - Frames are taken from `latest_event.entries[type=="exception"].data.values[-1].stacktrace.frames`, keeping only the last 15, which is the innermost.
  - A missing token (the factory returns `None`) gives 503 "Sentry (hosted) isn't configured on mom".
  - A 403 for prod without a grant is covered by Task 4.
- [ ] **Step 2: Implement both routes.** Both need `env` for scope, because an issue short id doesn't reveal its environment to the caller before the lookup. Both are audited.
- [ ] **Step 3:** Run the tests. Expected: PASS.
- [ ] **Step 4:** Commit with `feat(observability): env-scoped Sentry issue and search routes`.

### Task 7: Relax the rate limit on the flow routes

**Files:**
- Investigate first: the pilot got `429` with "Rate limit exceeded. Retry after 60 seconds" on `agent-ticket-flow/agent/*`, but no `@limiter.limit` appears in `agent_ticket_flow/`. Look for a SlowAPI `default_limits`, middleware, or an nginx ingress limit in kuberist (`bases/*/mom*` ingress annotations such as `nginx.ingress.kubernetes.io/limit-rpm`).
- Modify: whichever file sets that limit. Test: add a regression test where one exists; for an ingress annotation, rely on the kuberist manifest diff.

- [ ] **Step 1:** Find the source and record it in the commit message.
- [ ] **Step 2:** Exempt `/agent-ticket-flow/*` and `/observability/*` from that limit, or raise it for those paths, so that 15 parallel agents under one login don't hit it. The rest of mom stays as it is.
- [ ] **Step 3:** Verify with a loop of 60 `gravi-axi flow ls --limit 1` calls in 30 seconds against mom after it's deployed. Expected: no 429.
- [ ] **Step 4:** Commit with `fix(ticket_flow): stop the per-caller rate limit throttling agent fleets`.

## Part B: gravi-cli (crew: Bravo, `bestbuy_tools/mom/cli`)

### Task 8: `MomClient` observability methods

**Files:**
- Modify: `mom/cli/gravi_cli/client.py`, adding methods next to the flow methods at line ~1777
- Test: `mom/cli/tests/test_client.py`

**Interfaces:**
- Produces:
  - `obs_logs(token, env, filters, container=None, since="24h", limit=200, direction="backward") -> dict`
  - `obs_logs_count(token, env, filters, container=None, since="24h") -> dict`
  - `obs_sentry_issue(token, short_id, env) -> dict`
  - `obs_sentry_search(token, env, query, since="14d") -> dict`

- [ ] **Step 1: Write the failing tests** using `responses`.
  - Check the method, the path (`observability/logs/query` and the others) and the JSON body.
  - A 403 body `{"detail": "Requires observability:prod:jhoil for jhoil. Ask a mom admin."}` raises `PermissionDeniedError`, with that message kept.
- [ ] **Step 2: Implement the methods.** Each is one `self._make_request(...)` call.
- [ ] **Step 3:** Run `cd mom/cli && uv run pytest tests/test_client.py -v`. Expected: PASS.
- [ ] **Step 4:** Commit with `feat(cli): MomClient observability methods`.

### Task 9: `gravi-axi logs` and `gravi-axi sentry`

**Files:**
- Modify: `mom/cli/gravi_cli/axi.py`, adding two groups after the `flow` group at line ~1388
- Create: `mom/cli/gravi_cli/observability.py`, with pure formatting helpers in the style of `ticket_flow.py`
- Test: `mom/cli/tests/test_observability_cli.py`

**Interfaces:**
- Produces these commands:
  - `gravi-axi logs <env> [FILTERS] [--container C] [--since 24h] [--limit 200] [--json]`
  - `gravi-axi logs count <env> [FILTERS] [--container C] [--since 24h]`
  - `gravi-axi sentry issue <SHORT_ID> --env <env>`
  - `gravi-axi sentry search <env> <QUERY> [--since 14d]`
- Produces the helper `format_lines(resp) -> dict`, which gives a TOON-friendly table.

- [ ] **Step 1: Write the failing tests.** Use the `mc` fixture pattern from `test_ticket_flow.py`.

```python
def test_logs_passes_args_and_prints_toon(mc):
    mc.obs_logs.return_value = {"env": "sup", "namespace": "sup", "truncated": False,
                                "lines": [{"ts": "2026-09-29T10:00:00Z", "container": "backend-web", "level": "ERROR", "message": "KeyError: 'Northeast'"}]}
    res = CliRunner().invoke(axi.main, ["logs", "sup", '| level="ERROR"', "--since", "4h"])
    assert res.exit_code == 0
    mc.obs_logs.assert_called_once_with("tok", "sup", '| level="ERROR"', container=None, since="4h", limit=200, direction="backward")
    assert "KeyError" in res.output

def test_logs_permission_denied_exits_4_with_grant(mc):
    mc.obs_logs.side_effect = PermissionDeniedError("Requires observability:prod:jhoil for jhoil. Ask a mom admin.")
    res = CliRunner().invoke(axi.main, ["logs", "jhoil", ""])
    assert res.exit_code == axi.EXIT_PERMISSION and "observability:prod:jhoil" in res.output

def test_logs_truncated_prints_hint(mc):
    mc.obs_logs.return_value = {"env": "sup", "namespace": "sup", "truncated": True, "lines": []}
    res = CliRunner().invoke(axi.main, ["logs", "sup", ""])
    assert "# truncated" in res.output
```

- [ ] **Step 2: Implement.** Use `@main.group("logs", invoke_without_command=True)`, whose default action runs the query, and add a `count` subcommand. Add `@main.group("sentry")` with `issue` and `search`. Use `@json_opt` and `@handle_errors` as the flow commands do. When the response is truncated, print the hint `# truncated at <limit> lines; narrow the filter or --since`.
- [ ] **Step 3:** Run `uv run pytest tests/test_observability_cli.py -v`. Expected: PASS.
- [ ] **Step 4:** Commit with `feat(cli): gravi-axi logs and sentry commands`.

### Task 10: `gravi-axi flow doctor`

**Files:**
- Create: `mom/cli/gravi_cli/doctor.py`, with the check functions, each returning `Check`, plus `run(gate)`
- Modify: `mom/cli/gravi_cli/axi.py`, adding `@flow.command("doctor")`
- Test: `mom/cli/tests/test_doctor.py`

**Interfaces:**
- Produces:
  - `@dataclass class Check: name: str; status: Literal["ok","warn","fail"]; detail: str; fix: str; gates: tuple[str, ...]`
  - `def run(gate: str | None, repo: Path, runner=subprocess.run, mom=None) -> list[Check]`
  - `def blocked(checks, gate) -> bool`
  - The constant `REQUIRED_CLI = "0.20.0"`, which is the release this plan ships in.

- [ ] **Step 1: Write the failing tests.** Every external call goes through an injected `runner(cmd) -> CompletedProcess` and a fake `mom`, so the tests never touch the real machine. These cover Review Focus 4.

```python
from pathlib import Path
from gravi_cli import doctor

def fake_runner(results):
    def run(cmd, **kw):
        key = " ".join(cmd)
        for prefix, (rc, out) in results.items():
            if key.startswith(prefix):
                return subprocess.CompletedProcess(cmd, rc, out, "")
        raise FileNotFoundError(cmd[0])
    return run

def test_missing_claude_is_fail_not_traceback(tmp_path):
    checks = doctor.run("estimate", tmp_path, runner=fake_runner({}), mom=FakeMom())
    c = next(c for c in checks if c.name == "claude")
    assert c.status == "fail" and "install" in c.fix.lower()

def test_not_an_aio_checkout(tmp_path):
    checks = doctor.run("estimate", tmp_path, runner=fake_runner({"git remote get-url": (0, "git@github.com:acme/other.git")}), mom=FakeMom())
    assert next(c for c in checks if c.name == "aio_checkout").status == "fail"

def test_dangling_skill_symlink_is_fail(tmp_path):
    skills = tmp_path / ".claude/skills"; skills.mkdir(parents=True)
    (skills / "snd-flow-bugs-estimate").symlink_to(tmp_path / "gone")
    checks = doctor.run("estimate", tmp_path, runner=aio_runner(), mom=FakeMom())
    c = next(c for c in checks if c.name == "skills")
    assert c.status == "fail" and "snd-flow-bugs-estimate" in c.detail

def test_shadowed_gravi_is_fail_and_names_copy(tmp_path):
    r = fake_runner({"which -a gravi-axi": (0, "/home/u/.pyenv/shims/gravi-axi\n/home/u/.local/bin/gravi-axi\n"),
                     "/home/u/.pyenv/shims/gravi-axi --version": (0, "gravi-axi, version 0.16.0")})
    c = next(c for c in doctor.run(None, tmp_path, runner=r, mom=FakeMom()) if c.name == "gravi_cli")
    assert c.status == "fail" and ".pyenv" in c.fix

def test_sentry_prod_scope_missing_is_warn_for_triage(tmp_path):
    mom = FakeMom(obs_nonprod_ok=True, prod_keys=[])
    c = next(c for c in doctor.run("triage", tmp_path, runner=aio_runner(), mom=mom) if c.name == "observability_prod")
    assert c.status == "warn"

def test_blocked_only_on_fail_for_requested_gate():
    checks = [doctor.Check("git_rc", "fail", "", "", ("triage",))]
    assert doctor.blocked(checks, "triage") and not doctor.blocked(checks, "estimate")
```

- [ ] **Step 2: Implement the checks.** Each one catches `FileNotFoundError` and `subprocess.TimeoutExpired`, and those become a `fail` result.

| name | gates | check | fail or warn |
|---|---|---|---|
| `gravi_cli` | all | `which -a gravi-axi`, then `<first> --version` is ≥ `REQUIRED_CLI` | fail; the fix names the shadowing copy and gives `uv tool install --force gravi-cli==X` |
| `mom_login` | all | `mom.flow_list(token, limit=1)` succeeds | fail; the fix is `gravi login`, and a 403 means asking for the Ticket Flow agent role |
| `super` | all | `super read jira://KB-1/status` returns 0 | fail; the fix gives the `super` install steps |
| `claude` | all | `claude --version` returns 0 | fail; the fix gives the Claude Code install link |
| `aio_checkout` | all | `git -C repo remote get-url supply_and_dispatch_aio` or `origin` contains `supply_and_dispatch_aio` | fail; the fix is `--repo PATH` or a clone |
| `skills` | all | `.claude/skills/snd-flow-bugs-{estimate,triage}/SKILL.md` and `snd-flow-recruit/SKILL.md` all exist, following symlinks | fail, listing each missing or dangling skill |
| `git_rc` | triage | `git -C repo ls-remote <remote> RC` returns 0 | fail; the fix gives the GitHub access steps |
| `observability_nonprod` | triage | `mom.obs_logs_count(token, "sup", "", since="5m")` succeeds | fail; the fix is to ask for the Observability Reader role |
| `observability_prod` | triage | list the prod keys in the caller's scope; this needs a small route, `GET /observability/scope`, added to Task 4's router | warn when none; the fix is to ask an admin for a prod grant |
| `prod_data` | triage | `gravi query costco_test_cn backend order_v2 --count` returns 0 | warn |
| `burners` | triage | `gravi burner list` returns 0 | warn |

Add `GET /observability/scope` to the Task 4 router. It returns `{nonprod: bool, prod: ["jhoil", ...] | "*"}`, computed with `get_user_permissions`, and it needs a test.

Output: a TOON table with a row per check (`status icon, name, detail`). Every non-ok row is followed by `# fix: ...`. The exit code is 2 when `blocked(checks, gate)`, and 0 otherwise.

- [ ] **Step 3:** Run `uv run pytest tests/test_doctor.py -v`. Expected: PASS.
- [ ] **Step 4:** Commit with `feat(cli): gravi-axi flow doctor toolchain checks`.

### Task 11: `gravi-axi flow recruit`

**Files:**
- Modify: `mom/cli/gravi_cli/axi.py`, adding `@flow.command("recruit")`
- Create: `mom/cli/gravi_cli/recruit.py`, holding `lead_prompt(gate, parallel, count, minutes) -> str` and `launch(repo, prompt, runner=os.execvp)`
- Test: `mom/cli/tests/test_recruit.py`

**Interfaces:**
- Consumes: `doctor.run` and `doctor.blocked` (Task 10), `MomClient.flow_list` (existing)
- Produces: `gravi-axi flow recruit <estimate|triage> [--parallel 3] [--count 5] [--minutes 60] [--repo PATH] [--dry-run]`

- [ ] **Step 1: Write the failing tests.** These cover Review Focus 5.

```python
def test_recruit_blocked_by_doctor_does_not_launch(mc, mocker):
    mocker.patch("gravi_cli.recruit.doctor.run", return_value=[Check("claude", "fail", "not installed", "install Claude Code", ("estimate","triage"))])
    launch = mocker.patch("gravi_cli.recruit.launch")
    res = CliRunner().invoke(axi.main, ["flow", "recruit", "estimate"])
    assert res.exit_code == 2 and not launch.called and "install Claude Code" in res.output

def test_recruit_empty_queue_exits_0_without_launch(mc, mocker):
    mocker.patch("gravi_cli.recruit.doctor.run", return_value=[])
    mc.flow_list.return_value = {"tickets": []}
    launch = mocker.patch("gravi_cli.recruit.launch")
    res = CliRunner().invoke(axi.main, ["flow", "recruit", "triage"])
    assert res.exit_code == 0 and "nothing waiting at triage" in res.output and not launch.called

def test_recruit_launches_claude_in_repo_with_limits(mc, mocker, tmp_path):
    mocker.patch("gravi_cli.recruit.doctor.run", return_value=[])
    mc.flow_list.return_value = {"tickets": [{"key": "KB-1"}]}
    launch = mocker.patch("gravi_cli.recruit.launch")
    CliRunner().invoke(axi.main, ["flow", "recruit", "triage", "--parallel", "2", "--count", "4", "--minutes", "30", "--repo", str(tmp_path)])
    repo, prompt = launch.call_args.args
    assert repo == tmp_path
    assert "snd-flow-recruit" in prompt and "triage" in prompt and "parallel 2" in prompt and "count 4" in prompt and "minutes 30" in prompt

def test_dry_run_prints_command(mc, mocker):
    mocker.patch("gravi_cli.recruit.doctor.run", return_value=[])
    mc.flow_list.return_value = {"tickets": [{"key": "KB-1"}]}
    res = CliRunner().invoke(axi.main, ["flow", "recruit", "estimate", "--dry-run"])
    assert "claude" in res.output
```

- [ ] **Step 2: Implement it.**
  - `lead_prompt` returns `f"Run the snd-flow-recruit skill for gate {gate}: parallel {parallel}, count {count}, minutes {minutes}."`
  - `launch` changes into the repo and runs `os.execvp("claude", ["claude", "--permission-mode", "acceptEdits", prompt])`.
  - The waiting count comes from `mom.flow_list(token, next_gate=gate, limit=1)`.
  - Validation: `--parallel` must be between 1 and 8, and `--count` and `--minutes` must each be at least 1.
- [ ] **Step 3:** Run `uv run pytest tests/test_recruit.py -v`. Expected: PASS.
- [ ] **Step 4:** Commit with `feat(cli): gravi-axi flow recruit`.

## Part C: mom frontend (crew: Bravo)

### Task 12: The Recruit dialog

**Files:**
- Create: `mom/frontend/src/components/TicketFlow/RecruitModal.tsx` and `mom/frontend/src/components/TicketFlow/recruitCommand.ts`
- Modify: `mom/frontend/src/pages/TicketFlowPage.tsx`, adding the button to the top-bar `<Space>` at line ~305, after `flow-refresh`
- Test: `mom/frontend/src/components/TicketFlow/RecruitModal.test.tsx` and `recruitCommand.test.ts`

**Interfaces:**
- Consumes: `useTicketFlowBoard` for the waiting counts per gate (`next_gate` of estimation or triage, with no `held_by`)
- Produces: `recruitCommand({gate, parallel, count, minutes}) => string`

- [ ] **Step 1: Write the failing tests.**

```ts
import { recruitCommand } from "./recruitCommand";
it("omits defaults and includes changed limits", () => {
  expect(recruitCommand({ gate: "triage", parallel: 3, count: 5, minutes: 60 })).toBe("gravi-axi flow recruit triage");
  expect(recruitCommand({ gate: "estimate", parallel: 2, count: 10, minutes: 30 }))
    .toBe("gravi-axi flow recruit estimate --parallel 2 --count 10 --minutes 30");
});
```

- The modal test checks four things:
  - the button `data-testid="flow-recruit"` opens it
  - the gate picker shows "Triage (4 waiting)"
  - changing count updates the `data-testid="recruit-command"` text
  - the copy button calls `navigator.clipboard.writeText` with the command
- The doctor hint line reads "First time? Run `gravi-axi flow doctor triage`", together with `uv tool install gravi-cli`.
- [ ] **Step 2: Implement it** as an antd `Modal` (`footer={null}`, `width={620}`), in the style of `BuildAvailabilityModal.tsx`, with `Segmented` for the gate and `InputNumber` for the limits. Use theme tokens only.
- [ ] **Step 3:** Run `cd mom/frontend && yarn vitest run src/components/TicketFlow/RecruitModal.test.tsx src/components/TicketFlow/recruitCommand.test.ts`. Expected: PASS.
- [ ] **Step 4:** Commit with `feat(ticket_flow): Recruit dialog with generated command`.

### Task 13: Ship Parts A–C

- [ ] **Step 1:** Run the full backend suite with `cd mom/backend && uv run pytest -q`, the CLI suite with `cd mom/cli && uv run pytest -q`, and the frontend with `cd mom/frontend && yarn vitest run && yarn build`. Everything must pass.
- [ ] **Step 2:** Squash to one commit, run `git fetch bb_tools mom`, rebase onto `bb_tools/mom`, then run `git push bb_tools <branch>:mom`.
- [ ] **Step 3:** Report that a `gravi-cli-v0.20.0` tag is needed. The mate tags it.
- [ ] **Step 4:** Report the admin setup, which is a one-time job:
  - A Grafana service account with the Viewer role, and its token.
  - One internal integration per Sentry instance, with `event:read`, `project:read` and `org:read`.
  - All three tokens go into Google Secret Manager as `MOM_GRAFANA_SERVICE_TOKEN`, `MOM_SENTRY_HOSTED_TOKEN` and `MOM_SENTRY_SELF_HOSTED_TOKEN`, synced by kuberist `sync-secrets.py`. `MOM_GRAFANA_URL` also needs setting.

## Part D: AIO skills (mate, `supply_and_dispatch_aio/.claude/skills`)

### Task 14: The estimate and triage skills accept a pre-started run

**Files:**
- Modify: `snd-flow-bugs-estimate/SKILL.md` and `snd-flow-bugs-triage/SKILL.md`, both at step 1 ("Start your run")

- [ ] **Step 1:** Add this rule at the top of step 1 in both files:

  > **If you were told your run is already started (run id X), skip `flow start`.** Pass `--run X` on every write (`estimate`, `report`, `heartbeat` and `fail`). A lead that claimed the ticket for you has already opened the run.

- [ ] **Step 2: Verify.** Dry-run one estimate subagent with the prompt "estimate KB-44651; your run is already started, run id TEST". Its `commands.sh` must have no `flow start`, and every write must pass `--run TEST`.

### Task 15: Triage references move to `gravi-axi logs` and `gravi-axi sentry`

**Files:**
- Modify: `snd-flow-bugs-triage/references/sentry-and-logs.md`, `snd-flow-bugs-triage/references/environments.md` and `snd-flow-bugs-triage/SKILL.md`

- [ ] **Step 1: Rewrite `sentry-and-logs.md`.**
  - **Loki:** use `gravi-axi logs <key> '<filters>' --container backend-web --since 24h` and `gravi-axi logs count <key> '<filters>' --since 24h`. Keep the traps about `|~ "error"`, the 4-day window, the ~54-day retention, `| json` and the image-sha trick.
  - **Sentry:** use `gravi-axi sentry issue <SHORT_ID> --env <key>` and `gravi-axi sentry search <key> '<query>'`.
  - A 403 names the missing grant. Put that in the evidence field ("prod logs not available: missing observability:prod:jhoil") and continue.
  - Remove every MCP instruction.
- [ ] **Step 2: Shrink the crosswalk table in `environments.md`** to "ticket says → instance key". The namespace and Sentry columns go, because mom maps them now. Keep the `gravi query` and BBDClient sections.
- [ ] **Step 3: Update `SKILL.md`.** Step 2 no longer falls back to the Atlassian MCP for Jira, because doctor now requires `super`. The Sentry and Loki bullet in step 3 points at the new commands.
- [ ] **Step 4: Verify.** Dry-run one triage subagent on KB-51844 once Part A is deployed. It must query Loki and Sentry through `gravi-axi` and never call an MCP.

### Task 16: The `snd-flow-recruit` lead skill

**Files:**
- Create: `snd-flow-recruit/SKILL.md`

- [ ] **Step 1: Write the skill.** Its content:
  - **Input:** gate, parallel, count and minutes, taken from the launch prompt.
  - **The loop, which follows spec component 5:**
    1. Record the start time.
    2. While there's a free slot, fewer than `count` tickets have started, and fewer than `minutes` have elapsed: run `gravi-axi flow claim <gate> --name snd-flow-recruit --run-ref recruit-<start-ts> --json`. With no ticket, stop claiming.
    3. Otherwise start a background subagent (Agent tool, general-purpose) with the prompt: "Read `.claude/skills/<gate skill>/SKILL.md` and follow it for `<KEY>`. Your run is already started, run id `<run_id>`. Reply with the skill's one-line report."
    4. Wait at least 5 seconds between claims.
  - **As each subagent reports:** print one line (`✅ KB-X: <one-line report>` or `🔴 KB-X: <error>`), then fill the slot again.
  - **At the end:** wait for the running subagents, then print a summary table (key, outcome, duration) plus the totals.
  - **Rules:**
    - Never read ticket content yourself; pass keys only.
    - Exit 5 means wait 60 seconds and retry.
    - Don't retry a failed subagent, because its run's `fail` releases the ticket.
    - Don't write to Jira.
- [ ] **Step 2: Verify** once Part B is released, with `gravi-axi flow recruit estimate --parallel 2 --count 2`, run in the AIO checkout against the real queue. Two tickets should be estimated in parallel, each reported in one line, and then the run should stop.

### Task 17: The end-to-end bootstrap check

- [ ] **Step 1:** In a clean shell user, or with `HOME` pointed at an empty directory, install gravi-cli with `uv tool install gravi-cli==0.20.0` and run `gravi-axi flow doctor`. Every ❌ must print a fix that works when followed literally. Record any that doesn't.
- [ ] **Step 2:** Follow the fixes until estimation is clear, then run `gravi-axi flow recruit estimate --parallel 1 --count 1`. One ticket should be estimated.
