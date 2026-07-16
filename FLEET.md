# Fleet — in-flight work   ·   crew = live

Phases: Plan Build House Test Review Valid Merge Ship  (●done ◉now ○todo)

 Ticket     Summary                                                Project        Phase            Crew
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 SSRM-feas  SSRM paging feasibility                                snd_aio        Delivered        🟡 idle
   ↳ no ticket (research)                                                         ↳ report ready
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 KB-46734   BC alloc on mid-shift move                             snd_aio        ●●●●◉○○○ Review  🔴 feedback
   ↳ Ready for Review                                                             ↳ #1306
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 KB-41885   Sched audit — QA assist                                snd_aio        ●●●●●◉○○ Valid   🟡 idle
   ↳ captain validating (PR #1275)                                                ↳ /v1/ audit rows on diesel
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 KB-45195   Breadcrumbs on by default                              bestbuy_tools  ●◉○○○○○○ Build   🔄 working
   ↳ Part 3 rollout tooling                                                       ↳ prod flip script (dry-run)
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 KB-45174   Carrier Alloc Report                                   snd_aio        ●●◉○○○○○ House   🟡 idle
   ↳ In Progress                                                                  ↳ paused (limit)
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────

## 🔴 Needs feedback — captain
- **KB-46734** (KB-46734_breadcrumb-reassign) — AC-4 prod-query decision still pending (run read-only Sheetz breadcrumb query?); Energy Points estimate is captain's call

## Recently done
- OORM — OORM epic (Route Mileage & Out-of-Route Viz) shipped — PR #1197 merged to RC 2026-07-15 (Nathan). Includes KB-45857 (pre-trip odometer now anchors the first driving leg) + KB-47541 (simulator fabricates a monotonic odometer trail, OdometerSource.simulated). Verified live on the rig burner for QA (sheharyar). KB-45878 dropped (column not ready). Crew retired 2026-07-16; throwaway _demo_enable_variance.py preserved in scratchpad.
- Sentry — Sentry sweep triaged into the KB-47499 epic; captain owns the follow-up manually. Crew retired 2026-07-16. (Its worktree had been repurposed for KB-48262 store_tank_view cache-hit 500 fix — PR #1291, merged 2026-07-13.)
- KB-46984 — Inactive tank removed from manifold. PR #1289 merged to RC 2026-07-13 (Nathan). Two review flags left for follow-up: (1) retroactive prod data — the customer's existing bad row won't self-heal (forward-only fix); (2) AC-1 lone-survivor deviation (possible lead override). Jira transition pending (MCP was down).
