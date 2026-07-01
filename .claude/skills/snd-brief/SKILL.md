---
name: snd-brief
description: >-
  Dispatch a crewmate for an AIO ticket and run the mate↔crew conversation: compose a self-contained
  brief (task + worktree isolation + inlined how-to + definition-of-done), send it into the crewmate's
  herdr pane, then supervise via herdr and the FLEET STATUS report-back protocol — answer decisions,
  surface blocks to the captain, and collect the done handoff. Use right after snd-kickoff when handing
  a ticket to a crewmate — "dispatch KB-1234 to a crewmate", "brief the crew", "hand this ticket off",
  "spawn a worker for KB-XXXX", "send the crew its orders", "kick the crewmate off". Mate-tier (runs at
  cwd=frigate). AIO-specific.
---

# snd-brief — dispatch + converse with a crewmate

The mate never codes tickets itself — it briefs a crewmate in the worktree and supervises. This skill is
the **middle** of the dispatch loop: *compose → send → supervise → collect.* It follows `snd-kickoff`
(which made the worktree + provisioned local skills) and precedes the back-half ceremony (squash/push →
PR → Jira), which the mate owns — see the operating manual.

**Why the brief must be self-contained:** the crewmate runs at `cwd=<worktree>` — a snd_aio checkout —
so it auto-loads the repo's own `CLAUDE.md` and skills, but it **cannot** open frigate's docs. Don't
point it at `frigate/docs/...`; carry everything it needs inline.

## 1. Compose the brief

Fill this template with the ticket + worktree. Keep it tight — the crew already has the repo's
`CLAUDE.md` + skills; the brief adds the task, the guardrails, and the protocol.

---
**Task — KB-XXXX: \<summary\>**  (https://gravitatedxp.atlassian.net/browse/KB-XXXX)
\<1–3 lines: what to build/fix. Note small-bug vs feature.\>

**Where you are.** You're in an isolated worktree for this ticket — branch `KB-XXXX_descriptor` off
`RC`. Work only here; don't touch other worktrees or the main checkout. Your repo `CLAUDE.md` and skills
(`housekeeping`, `backend_testing`, and the frontend skills if you touch `frontend/`) are in context —
follow them.

**How to work.**
- Non-trivial? Spec the approach first, then implement. Small bug? Go straight to the fix.
- Follow the repo conventions (i18n wrapping, the datetime rules, typed FE client, etc. — your `CLAUDE.md`).
- When the code is done, run the **`housekeeping`** skill (simplify → review → BE/FE compliance → autofix).
- Tests: `uv run pytest` for backend; add/adjust tests per `backend_testing`.
- Commit your work to the branch in tight, reviewable commits.

**Definition of done.** Branch implemented, housekeeping clean, tests green, committed. **Do NOT** push,
open a PR, or touch Jira — the mate owns that ceremony. Your job ends at *branch ready + reported*.

**Report back — the FLEET STATUS protocol.** I'm the **mate**, watching you through herdr — not a human
at a keyboard. Don't stall on open-ended questions. When you finish, get blocked, or need a decision,
**end your turn with exactly this block** (then stop), so herdr marks you done/idle and I can read it:

```
=== FLEET STATUS ===
ticket: KB-XXXX
state: done | blocked | needs-decision | failed
summary: <one line — what you did, or where you're stuck>
question: <only if blocked/needs-decision — the specific decision I must make>
handoff: <only if done — branch, test result, anything the ceremony needs>
=== END ===
```
I'll either answer (and you continue) or take it from here.
---

## 2. Send it

If `snd-kickoff` didn't already start the agent, spawn it, then deliver the brief into the pane:

```bash
herdr agent start KB-XXXX --workspace <snd_aio-ws> --cwd <worktree> -- claude
herdr agent send <pane> "<the composed brief>"
```

Update the board: the ticket's row → stage `Build`, crew 🟢.

## 3. Supervise + converse (the back-and-forth)

Block on the crew's state, read its FLEET STATUS block, and respond:

```bash
herdr wait agent-status <pane> --status done --timeout <ms>   # also wakes on blocked/idle
herdr agent read <pane> --source recent --lines 60            # read the FLEET STATUS block
```

Act on `state:`:
- **done** → sanity-check the `handoff`, then move to the ceremony (squash+push → `snd-jira-housekeeping`
  gate → draft PR). Board → `PR`.
- **needs-decision** → if it's yours, `herdr agent send <pane> "<decision>"` and `wait` again. If it's
  the captain's call, surface it (board **Blocked** + a ping) and hold.
- **blocked** → resolve with a one-line steer if you can, else surface to the captain. Put the crew's
  `question:` in the board's Blocked section.
- **failed** → read the evidence, report to the captain, decide retry vs. hand back.

Keep the loop tight — **answer → `herdr agent send` → `herdr wait` again**. Short steers down, FLEET
STATUS blocks up. That's the conversation.

## Notes

- If the crew asks *without* the block (just chats at you), still read + answer via `herdr agent send`,
  but nudge it back to the protocol so future signals are machine-clear.
- One crewmate per ticket; run several concurrently across worktrees and supervise each.
- The definition of done is **mode-shaped**: this is AIO's draft-PR flow (crew stops at branch-ready). A
  direct-PR / local-only project would fold the push/PR step into the crew's DoD instead.
