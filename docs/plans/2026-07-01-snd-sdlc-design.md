# SND SDLC — shareable crew capability (design)

**Date:** 2026-07-01 · **Status:** approved, in build

## Goal / reframe

A crewmate operating **inside the SND repo** should have everything it needs to carry a KB ticket
through the *entire* SDLC, launchable by a **command inside SND** — so the capability is **shareable to
teammates** who have neither frigate, herdr, nor the mate. The mate remains Nathan's personal
supervisor layer that *dispatches this same SND capability* and reads its status; it is secondary and
refined over time. **Center of gravity moves from frigate → the SND repo.**

Key finding that shaped this: **SND is not greenfield.** It already ships a team-tracked Claude toolkit
on clone — commands (`/commit`, `/hoist`, `/post-tests`, `/psp`, the `/qa-*` E2E chain) and skills
across `backend/`, `frontend/`, `automation/`. So the work is **complete + connect**, not invent.

## Architecture decision: dedicated skills + thin orchestrator (later)

Chosen after weighing autonomous-orchestrator vs. guided-state-machine vs. skills-only:

- **Foundation (build first): solid, standalone, independently-invocable phase skills.** This is the
  load-bearing, robustness-defining choice, common to every option, and fits the repo's one-command-
  at-a-time culture.
- **Composition layer (build last): a *thin* orchestrator** — a *sequencer + gatekeeper + reporter*
  that **delegates to skills and never re-implements** their logic. Fat orchestrators are the failure
  mode of composition; keep all the "how" in skills, only the "what next / may I proceed" in the
  orchestrator.
- Style: **phase-gated with report-back** (emit a FLEET-STATUS-style block at each phase boundary),
  **hard-stopping at the human gates** (non-draft PR, Validation/QA, Merge) which exist in every option
  anyway. Auto-advance vs. pause is a **mode flag** (auto for a dispatched crew; pause for a dev), not a
  different design.
- **Sequence the work pieces-first** so the orchestrator is the last, cheapest, most-informed thing we
  build — and the pieces deliver value immediately (usable standalone + composable by the mate today).

## The skill set

| Phase | Owning skill | Status | Does / composes |
|---|---|---|---|
| Planning | `snd-kickoff` | NEW in snd (herdr-free) | Jira readiness read → branch off `RC` → *In Progress* → light spec; hands to `brainstorming` if installed, else specs inline |
| Building | *(none)* | REUSE | repo `CLAUDE.md` + FE/BE skills + `backend-i18n` + `/commit` |
| Housekeeping | `snd-housekeeping` | GRADUATE (local→shared) | simplify → review → BE/FE compliance table + autofix (already built) |
| Testing | `snd-testing` | GRADUATE (local→shared) | routes to `/qa-gen`/`/qa-heal`, `backend_testing`, burner via `gravi-burners` (already built) |
| Review | `snd-pr` | NEW | draft PR per `.github/pull_request_template.md` + `CLAUDE.md §PRs`; plain push (never `/hoist`); draft-only unless approved |
| Jira (cross-cut) | `snd-jira` | MIGRATE from frigate `snd-jira-housekeeping` | readiness gate (≥2 Dev/Validate, sprint, energy pts) + status moves + AC-drift → Dev Implementation; **composes `/post-tests`** for the coverage comment |
| Validation | *(human QA)* | GATE | `snd-jira` sets *Ready for Testing*, hands off |
| Merge | `snd-merge-check` | NEW, thin (or fold into orchestrator) | read-only: confirm CI green / no conflicts / not behind `RC` (never rebases, pushes, or merges) |
| Shipped | *(captain merges)* | GATE | — |

Reused as-is: `/commit`, `/post-tests`, `/qa-*`, `pytest-runner`, `e2e-gen-workflow`, `backend_testing`,
`api-patterns`, `grids`, `i18n`, `paginated-grid`, `backend-i18n`, `gravi`. (`/hoist` exists as a team
command but is **owner-manual — the pipeline never rebases or force-pushes.**)

## Locked decisions

1. **Naming:** `snd-` prefix for the pipeline skills; team domain skills stay unprefixed.
2. **Shareability posture:** degrade gracefully — reference `superpowers` (brainstorming / subagent-dev)
   only *if present*, with an inline fallback. Atlassian MCP + `gravi` documented as team prereqs.
3. **Relocate the SDLC playbook** out of frigate into snd (`docs/sdlc.md`) as the self-contained,
   herdr-free authority the skills point to.
4. **Kickoff/herdr split:** shared `snd-kickoff` is herdr-free (branch + Jira + spec) and branch-aware;
   the mate still creates the herdr worktree before dispatch.
5. **Scope flip:** these skills graduate to **committed/shared** (reversing the local-only default).

## Conflicts to respect
- SND `/commit` **forbids** `Co-Authored-By`/AI references (frigate's default adds them). Crew follows
  **SND's** convention when committing in snd.
- No hard dependency on **herdr** (teammate uses plain `git checkout -b`) or the **superpowers plugin**.

## Mate integration / migration
- `snd-brief` (mate↔crew dispatch + FLEET STATUS) **stays in frigate** — it's a mate concern.
- frigate's `snd-kickoff` / `snd-jira-housekeeping` are **superseded** by the snd `snd-kickoff` /
  `snd-jira`; the crew (in the worktree) runs the snd versions. frigate's mate-orchestration doc keeps
  only the herdr/workspace/dispatch layer and points at snd's `docs/sdlc.md` for the SDLC itself.
- Fleet board / phase track (frigate) unchanged — it's the mate's semantic overlay.

## Build order
Playbook → `docs/sdlc.md` (snd) → graduate `snd-housekeeping` + `snd-testing` → migrate `snd-jira`
(compose `/post-tests`) → new `snd-kickoff` → new `snd-pr` → thin `snd-merge-check` → **then** the
orchestrator.

## Deferred
- Orchestrator command shape (thin, phase-gated, auto/pause mode) — after the pieces are proven.
- `snd-merge-check` as its own skill vs. an orchestrator step.
- Retiring frigate's superseded mate-tier skills once the snd versions land.
