# Sentry Cleanup — Working Notes (2026-06-25)

Project: **capspire / best_buy_services** (region us.sentry.io). Working notes only — not for commit.

---

## SWEEP 2026-07-07 (assessment-only run — reconcile vs KB-47499, no mutations)

Access this run: no Sentry MCP in session → hit the Sentry REST API through the user's
authenticated Chrome (browser-connect, project id `5726180`). Epic KB-47499 now has 16 children
(added since 6/25: **KB-47528** tank create_endpoint @logger.catch mask — Done). Done: 47503, 47504,
47505, 47506, 47507, 47510, 47528. Still To Do: 47501, 47502, 47508, 47509, 47511, 47512, 47513,
47514, 47515.

### Reconciliation flags — tracked but Done yet still firing (NOT new tickets; verify deploy)
- **16S8** (KB-47506 *Done*) still firing **1,985** events, lastSeen 7/07 — fix not deployed or incomplete.
- **1F7Y** (KB-47504/47528 *Done*) **regressed → 5,441** — tank create_endpoint failing again.
- **1FTW** (KB-47510 *Done*) regressed → 121 ("Unable to find location matching id" — may be expected data miss).
- **1FQE** was mapped to KB-47510 (*Done*) but its **current** signature is a `None − datetime` TypeError
  in `get_in_cab_alerts_for_site`, not the TTLCache KeyError → effectively untracked (see new #2).

### NEW tickets FILED 2026-07-07 (captain approved — created under KB-47499)
1. **KB-47995** [Bug] distance_cache route `save_with_audit` — 1GG8+1GJ8+1GJA+1GHG (~959). `insert()`→DuplicateKeyError
   (`marathon_backend.route unique_by_…`), revisioned `save()`→RevisionIdWasChanged (same conflict, update
   path via `system_refresh` + /order/supply_options), + `CollectionWasNotInitialized` in `system_refresh`
   (Beanie Route/RouteAudit not init in scheduled-refresh ctx). Fallout from effective-dated route/RouteAudit
   work; exactly the failure mode backend CLAUDE.md warns about. `distance_cache/model.py:608 save_with_audit`, `:1025 system_refresh`, `:434 _persist_new`.
2. **KB-47996** [Bug] in_cab alert age `datetime − None` — 1GG6 (500) + 1FQE (360) ≈ 860. `resequence_order_list`→
   `get_all_retain_alerts` and `site_details_inventory`→`get_in_cab_alerts_for_site` subtract a None
   timestamp. (Corrects 1FQE off KB-47510.)
3. **KB-47997** [Bug] tank create_endpoint empty-string→float — 1GJN (6,546) `safe_float('')` ValueError; distinct from
   KB-47504 (NaN→JSON) & KB-47528 (mask). Pairs with 1F7Y regression (5,441) — verify 47504/47528 deploy.
4. **KB-47998** [Bug] actors.update_demand_profile_actor None.identity — 11AC (189, regressed, firstSeen 7/02).
5. **KB-47999** [Bug] actors.price_import_function `'float'.casefold` — 1GAB (146). String op on numeric import cell.
6. **KB-48000** [Bug] order/cancel_order IndexError in rescind email — 1FVF (135). `send_email/api.py:email_rescind_from_order`→`send_mail_to_carrier` indexes empty list.
7. **KB-48001** [Bug] store/store_tank_view StoreTankView ValidationError — 1FZ1 (130).

Fold / low-pri (NOT filed — captain call): 1FSC order/create_manual ZeroDivision (118) → **KB-47512**;
1GJP in_cab `process_drop` → `NotFoundError: No drop for store 328` (84) → **KB-47512** / relate KB-47503;
1FQH driver_schedule/assign None.flags (74) → small.

### Noise still flowing (existing filters look INEFFECTIVE; durable fix = KB-47501 still To Do)
`*Unable connect to node*` filter is not dropping **13SX** (10,266 events, lastSeen today) → verify the
inbound filter was actually saved/active. New noise-glob candidates:
- `*NOT_LEADER_FOR_PARTITION*` (1ESS /breadcrumb/pub) · `*NodeNotReadyError*` (1ESR, 14JD)
- `*Redis is loading the dataset in memory*` (1G2K, 10S3 /healthz) · `*connecting to redis-svc*` / `*reading from redis-*` (16Q6, 16MV, 1G2G, QAV)

Borderline (prior-run verdicts still pending, not re-triaged): W7E, WAV, 1F3E, 12Z5.

---

## Jira (epic KB-47499)
| Ticket | Type | Covers (Sentry) |
|---|---|---|
| KB-47499 | Epic | — (parent) |
| KB-47501 | Task | durable before_send + ignore_logger + OTEL endpoint |
| KB-47502 | Bug | 1E71 + 1DA3 (pricing archived curves) — design decision open |
| KB-47503 | Bug | 17KX (driver_pre_trip) |
| KB-47504 | Bug | 1F8K + 1F7Y (tank NaN) |
| KB-47505 | Bug | 1F73 (inventory WriteError tanks) |
| KB-47506 | Bug | 16S8 (ReasonCode validation) |
| KB-47507 | Bug | 1G9W (user_tasks int None) |
| KB-47508 | Bug | 1FEB (bol None compare) |
| KB-47509 | Bug | 1FHA + 1FHB (store upsert) |
| KB-47510 | Bug | 1FV5,1FZH,1G0N,1FV0,1FTX,1FTW,1FQE (LocationView TTLCache) |
| KB-47511 | Bug | 1GFB,1FRB,1FQC,1FKT (input validation → 422) |
| KB-47512 | Bug | 1E7Q,1FSB,1FRJ,19SC (order longtail) |
| KB-47513 | Bug | 1GGN (forecast redis pool) |
| KB-47514 | Task | 1EJD (bare traceback observability) |
| KB-47515 | Bug | 1DWS (worker None.messages) |

## Actions taken so far
- **Inbound Filters** (project → Custom Filters → Error Message + Log Message) added for transient infra noise — drops at ingest, no deploy. Patterns cover: OTEL exporter (`*Max retries exceeded with url: /v1/traces*`), redis `connecting to redis-big-svc`, httpx `ConnectError/ReadError`, `ClientDisconnect`, worker-init/shutdown/websocket `RuntimeError`s, SFTP, Kafka (`*Unable connect to node*`, `*KafkaConnectionError*`).
- **Archived (ignore-forever) 14 confirmed-noise issues**: 1ET3, 1FJN, 1ET4, 1FMS, 1G1P, 1F4M, 1G25, 1EPY, 1EF0, 1F7M, 1FQ2, 1GAK, 1G2Z, 1C4T.
- **Pending archive approval**: 13SX (Kafka, 1.05M), 14JD (KafkaConnectionError, 67k) — filter now drops new events regardless.
- **Pending your verdict (borderline noise vs real)**: W7E, WAV, 1F3E, 12Z5, 1GGN.

## Durable noise fix (TODO — needs deploy)
- Rewrite `backend/sentry_config.py:before_send` to walk the **whole** exception chain and match by type name (current single-frame `values:[{type}]` match misses chained errors).
- `ignore_logger(...)` for the OTEL exporter + aiokafka loggers.
- Backport `before_send` to services missing it: `auth`, `ims`, `forecast`.
- Fix OTEL collector endpoint config (`alloy-receiver` / `otel-collector` unreachable) — restores tracing.

---

## DOCUMENTED, SOLVE DEFERRED: pricing `effective_prices_many` KeyError (#1E71 + #1DA3 ≈ 335k)

**Root cause:** `pricing/price/endpoints.py:258` does `lift_method_lkp[curve_id]`; `lift_method_lkp` is built from live `flat_price` only, so it KeyErrors (500) for any curve_id with no live price. The 500 propagates to the backend as `#1DA3 ServiceException`.

**Why curves have no live prices:** `pricing/history/api.py:archive_old_prices` deletes `flat_price` docs whose `effective_to` is >45 days past (moves them to `price_history`). `backend/order/flag.py` reprices orders (incl. **completed** ones) as of their historical `bol_date`, requesting curves whose prices were archived.

**Prod data (mav):** 499k orders; 1,199 live curves; orders reference 1,746 curves; **883 (~51%) have zero live prices**; **82,649 orders (16.6%) reference a missing curve — 94% `complete`**. Top offenders date to May 2023. Event's curve `67b7…869`: flat_price=0, price_history=14, curve_group=active.

**Requirement:** user confirmed repricing completed orders **is important** → cannot just `.get()` (would silently zero historical prices). Needs a real `price_history` fallback.

**Open decision before implementing (the blocker):** the reprice writes `price, effective_from, effective_to, price_id, net_or_gross_type, contract_lifting_valuation_method` onto the order, but `price_history` only kept `price_id, effective_from, effective_to, value` — it **dropped `net_or_gross_type` and `contract_lifting_valuation_method`** (stored per-price in `flat_price`, not on the curve).
- Going forward: enrich `archive_old_prices` to keep all needed fields (easy).
- Already-archived (~131k refs across 883 curves): those two fields are **gone**. Decide either (a) treat them as constant-per-curve and recover from the order's existing component, or (b) accept degraded fidelity on those two fields for pre-archive history. *Needs a quick `flat_price` check: are those two fields constant within a curve_id?*

Also note: `archive_old_prices` claims it "won't delete the last price for a curve" yet 883 curves have zero live prices — a separate anomaly worth a follow-up.

---

## Remaining bugs to tackle — prioritized inventory

### Tier 1 — high volume
| Issue | Error / culprit | ~Events | Notes |
|---|---|---|---|
| 17KX | `UnableToProcessAllUpdatesError: missing driver_pre_trip` — in_cab.device_processing | 86,215 | Highest non-pricing. Real processing gap or expected-missing data? Needs a look. |
| 1F8K | `ValueError: nan not JSON compliant` — tank_inventory.readings_by_store_tank | 60,696 | NaN reading → JSON. Sanitize NaN→null. |
| 1F7Y | `ResponseValidationError` — tank_inventory.create_endpoint | 56,317 | Response/schema mismatch (likely same NaN/missing field). |
| 16S8 | `ValidationError for ReasonCode` — order.reason_code.schema.get_reason_codes | 51,466 | A persisted reason_code fails read validation. Data/schema drift. |
| 1F73 | `WriteError: path 'tanks' must exist` — store.model_v2.apply_inventory_update | 39,709 | Array update on missing `tanks`. Guard/upsert. |
| 1G9W | `TypeError: int(None)` — user_tasks.api.complete_fan_out_chunk | 33,116 | None where int expected. |

*(Tank cluster 1F8K+1F7Y+1F73 ≈ 156k may share a root area — fix together.)*

### Tier 2 — medium
| Issue | Error / culprit | ~Events | Notes |
|---|---|---|---|
| 1FEB | `TypeError '<' None/None` — bol.api.get_actual_end_from_order | 17,866 | Missing null guard in comparison. |
| 1FHA | `ValidationError invalid email` — /store/upsert_many | 10,968 | Bad email in store import → 422 not 500. |
| 1FHB | `AttributeError None.get` — /store/upsert_many | 3,544 | Same endpoint, null handling. |
| 1EJD | bare `Traceback (most recent call last):` | 5,162 | Poorly captured (no type) — fix instrumentation to even identify it. |
| 1DWS | `AttributeError None.messages` — __main__.main | 4,975 | Worker/main loop null. |

### Cluster — in_cab/driver_tracking `LocationView` KeyError (ONE root cause, many issues)
`get_geofence_from_location` is `@cached` on the whole LocationView; KeyError thrown inside `cachetools.TTLCache.expire` = unsynchronized concurrent TTLCache. One fix collapses: **1FV5, 1FZH, 1G0N, 1FV0, 1FTX** (~6.2k). Related location-lookup bugs in same endpoints: 1FTW (1,597), 1FQE (1,485).

### Cluster — input validation → 500 that should be 422 (small, easy wins)
1GFB `InvalidId ''` /order/route_log (1,212) · 1FRB `KeyError ''` /driver_schedule/move (750) · 1FQC `KeyError` /order/supply_options (1,293) · 1FKT `KeyError` bol.valuation (556).

### Order-domain longtail
1E7Q ValueError drop detail (2,491) · 1FSB AttributeError None.trailer_config /order/create_manual (854) · 1FRJ StopIndexError load step (1,141) · 19SC AutoReconciliationError (1,522).

### Borderline — need your verdict (noise vs real)
W7E `Failed to connect to monitor` tank proxy (69k) · WAV dramatiq coroutine timeout (54k) · 1F3E `TimeLimitExceeded` (4k) · 12Z5 `E-Sticking Unavailable` (9.5k) · **1GGN `ConnectionError: max number of clients reached` — actors.update_store_forecast (611, NEW)** — this one smells like a real redis/DB pool-exhaustion leak, not transient.
