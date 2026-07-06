# Plan-mode design gate for dispatched crews (design)

**Date:** 2026-07-06 · **Status:** approved, implementing

## Problem

Dispatched crews were skipping design — starting to code without a spec. Root cause was *not* skill
availability (`superpowers`/`brainstorming` is enabled at **user scope**, so the crew has it). It was two
things we built:

1. **`snd-kickoff` made design optional** — "*if the `brainstorming` skill is available, use it… otherwise
   a brief plan inline*" and "*small bug: skip the spec*." Bug-shaped tickets correctly skipped it.
2. **`brainstorming` needs a human to talk to.** It's a synchronous one-question-at-a-time interview with
   an approval gate. An auto-dispatched crew has the mate as a per-phase *continue* button, not a design
   collaborator — and the FLEET STATUS channel is coarse, turn-ended status, not a live dialogue. So even
   when it fired, it couldn't run.

The mate *is* reachable (crew↔mate over herdr), so "talk to the mate about design" is the right instinct —
but via the report-back channel, relayed to the captain (the design authority), not by running
brainstorming's interview over a high-latency relay.

## Decision: use Claude Code plan mode as the design gate

Plan mode is an **enforced rail, not an instruction** — read-only until a plan is approved, so the crew
*can't* jump to code. This fixes the problem at the mechanism level instead of the wording level.

### Verified (herdr test, isolated scratch workspace)

- `claude --permission-mode plan` boots straight into plan mode (read-only). Launch-flag force works.
- The crew produces a plan and **blocks** at ExitPlanMode — herdr `agent_status = blocked`, which the
  mate's existing background `wait agent-status --status blocked` already keys on. No new plumbing.
- The plan is persisted to `~/.claude/plans/<slug>.md` — a clean markdown artifact the mate reads and
  relays (no pane-scraping).
- The mate can drive the approval over herdr: `pane send-keys <pane> Enter` selected "Yes, and use auto
  mode"; the crew exited plan mode, made the edit, and continued autonomously. Proven with a real edit.
- The gate offers a redirect path (option 4, "Tell Claude what to change") — same `send-keys` +
  `send-text` mechanic, used for captain feedback.

## Design (approach A′ — rail always fires, mate absorbs trivial)

**Two paths, by who runs it:**
- **Dispatched crew** → `snd-brief` launches `claude --permission-mode plan`. Forced rail.
- **Interactive human** → can't be force-gated from a skill (they own the session); `snd-kickoff` *tells*
  them to design first — `brainstorming` (collaborative) or plan mode (solo). Unchanged from before.

**Crew flow — `snd-kickoff` reorders so design comes first:**
1. Resolve ticket + read AC/code + run Jira readiness *checks* (all reads — fine in plan mode).
2. Draft the implementation plan → **ExitPlanMode** (the gate). Crew goes `blocked`; plan written to
   `~/.claude/plans/*.md`.
3. **Only after approval:** create the branch, move Jira → In Progress, build. Nothing mutates until the
   design is blessed.

**Mate gate-handling (new branch in `snd-brief` supervise):** on `blocked`, the mate reads the pane. If
it's the ExitPlanMode prompt, it triages the plan:
- **Routine/trivial** (criteria below) → **auto-approves**: `send-keys <pane> Enter` → crew proceeds in
  auto mode. Recorded on the board (e.g. substep `plan auto-ok`).
- **Substantive** → **relays** the plan (`~/.claude/plans/*.md`) to the captain; on approval sends
  `Enter`, on redirect selects option 4 and sends the captain's feedback as text.
- A `blocked` that carries a FLEET STATUS `needs-decision` block (not the native plan prompt) is handled
  as today — the mate distinguishes by reading the pane.

**Auto-approve criteria (mate):** only when the plan is single-file-ish, adds no new model/endpoint/
pattern, changes no user-facing behavior, and matches existing conventions. Anything with a real design
choice, schema/API change, security/perf sensitivity, or ambiguous AC → escalate to the captain. So the
captain only ever sees non-trivial plans; the mate absorbs the trivial ones, transparently (recorded).

## Files

- `frigate/.claude/skills/snd-brief/` — launch in plan mode; brief notes the plan-first start; the
  gate-triage supervise branch.
- `SND/.claude/skills/snd-kickoff/` — reorder (design gate first, then branch/Jira/build); keep
  `brainstorming` for the interactive path.
- `SND/.claude/skills/snd-sdlc/` — note that Planning runs behind the plan gate.
- `SND/docs/sdlc.md` — Planning phase note.
- `frigate/CLAUDE.md` — dispatch-loop: the mate's plan-gate handling.

## Known limitation (documented, not solved)

A **resumed** crew (herdr restart / re-dispatch mid-ticket) isn't freshly launched in plan mode, so the
rail applies only to the initial dispatch. Resumes rely on the crew already being past Planning (its
cold-start detection in `snd-sdlc` re-anchors from branch/PR/Jira). Acceptable: the gate's job is to stop
a *fresh* crew from coding before it designs; a resumed crew already has an approved plan behind it.
