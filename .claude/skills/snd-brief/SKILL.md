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
(`housekeeping`, `backend_testing`, `snd-testing`, and the frontend skills if you touch `frontend/`) are
in context — follow them.

**How to work.**
- Non-trivial? Spec the approach first, then implement. Small bug? Go straight to the fix.
- Follow the repo conventions (i18n wrapping, the datetime rules, typed FE client, etc. — your `CLAUDE.md`).
- When the code is done, run the **`housekeeping`** skill (simplify → review → BE/FE compliance → autofix).
- Tests: `uv run pytest` for backend (per `backend_testing`); `yarn test` for FE unit. **Then run
  `snd-testing`** — it makes the explicit call on whether the change needs an E2E and, if so, authors it
  via `automation/snd_e2e`'s human-gated `/qa-start` chain (the E2E lands in the same PR).
- Commit your work to the branch in tight, reviewable commits.

**Definition of done.** Implemented, housekeeping clean, committed — then **pushed and validated**:
push your branch (you're *expected* to — CI needs the push, and a burner boots from the CI-built image),
watch CI go green, and verify the change (unit/E2E; a burner via `gravi-burners` when it warrants
real-app testing). The one hard gate: **never open a non-draft PR** — that's the captain's call (a
*draft* PR is fine). Your job ends at *pushed, CI green, validated, reported*; the mate/captain own the
non-draft PR + Jira transitions.

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

## 2. Pre-flight, then send

Three checks before you spawn — each one bit us once:

**a. Does the worktree still have a workspace?** herdr can drop a worktree's workspace on restart while
the git worktree survives (the board keeps showing the row because it joins on **worktree path**, not
workspace id). If `herdr workspace list` has no workspace whose `worktree.checkout_path` is this
worktree, recreate one on it — don't spawn into a stale/absent one:

```bash
herdr workspace create --cwd <worktree> --label <ticket-descriptor> --no-focus
# parse result.workspace.workspace_id and result.root_pane.pane_id from the JSON
```

**b. Are the crew skills the brief names actually provisioned?** `snd-kickoff` provisioned them *at
kickoff time* — but a skill added to the repo since (or a worktree older than the skill) leaves gaps.
Re-run kickoff's provision loop against the worktree so any new local-only skills get symlinked in, then
verify: `ls <worktree>/.claude/skills` shows `housekeeping`, `backend_testing`, etc.

**c. Spawn — and ALWAYS pin the workspace.** Without `--workspace`, `herdr agent start` splits the agent
into whatever pane is **focused** — i.e. the mate's own tab, not the worktree. Cleanest is to run the
agent as the workspace's root pane (matches the fleet's one-workspace-per-worktree convention):

```bash
herdr pane run <root_pane> "claude"          # <root_pane> from (a)
# or into an already-open project workspace:  herdr agent start KB-XXXX --workspace <ws> --cwd <worktree> -- claude
```

> **Skills load at startup.** If (b) added a skill *after* the agent was already running, it won't see
> it. Restart the pane: type `/exit` (Ctrl-C only clears the input; herdr rejects `C-d`), then
> `herdr pane run <pane> "claude"` again.

Now deliver the brief. `herdr agent send` writes literal text but does **not** press Enter — submit
separately. A multi-line brief is easiest from a file (backticks/newlines survive intact):

```bash
herdr agent send <pane> "$(cat /tmp/brief-KB-XXXX.md)"
herdr pane send-keys <pane> Enter
```

Update the board: the ticket's row → stage `Build`, crew 🟢 (auto once herdr sees the agent working).

## 3. Supervise + converse (the back-and-forth)

**herdr doesn't push state changes at you — you only learn a crew finished if you're waiting on it.**
So the instant you dispatch (or send a steer), arm the wait — and run it in the **background** so the
finish re-invokes you while you stay free to drive other crews and the captain. Never leave a working
crew unwatched; an unwatched finish gets missed (and the captain ends up noticing for you).

```bash
# run_in_background: true — you get re-invoked when the crew hits done/idle/blocked
herdr wait agent-status <pane> --status done --timeout 1800000
herdr agent read <pane> --source recent --lines 60            # then read its FLEET STATUS block
```

Running several crews? One background wait per pane. A *foreground* wait only makes sense when there's
nothing else to do until this single crew finishes.

Act on `state:`:
- **done** → sanity-check the `handoff`, then move to the ceremony (squash+push → `snd-jira-housekeeping`
  gate → draft PR). Board → `PR`.
- **needs-decision** → if it's yours, `herdr agent send <pane> "<decision>"` and `wait` again. If it's
  the captain's call, surface it (board **Blocked** + a ping) and hold.
- **blocked** → resolve with a one-line steer if you can, else surface to the captain. Put the crew's
  `question:` in the board's Blocked section.
- **failed** → read the evidence, report to the captain, decide retry vs. hand back.

Keep the loop tight — **answer → `herdr agent send` → re-arm the background `herdr wait`**. Short
steers down, FLEET STATUS blocks up. That's the conversation.

## Notes

- If the crew asks *without* the block (just chats at you), still read + answer via `herdr agent send`,
  but nudge it back to the protocol so future signals are machine-clear.
- One crewmate per ticket; run several concurrently across worktrees and supervise each.
- The definition of done is **mode-shaped**: in AIO the crew pushes, gets CI green, and validates (incl.
  burners) — pushing is theirs, and the captain's *only* gate is a **non-draft PR**. A direct-PR /
  local-only project would shift where that gate sits instead.
