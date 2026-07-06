---
name: snd-brief
description: >-
  Dispatch a crewmate to run the AIO SDLC end-to-end and supervise it. Set up an isolated worktree on the
  feature branch, launch a crewmate that drives the ticket via the `snd-sdlc` skill (auto mode), then converse via herdr
  and the FLEET STATUS report-back protocol — answer decisions, surface blocks to the captain, collect the
  done handoff (draft PR + Ready for Review). Use when handing an AIO ticket to a crewmate — "dispatch
  KB-1234", "brief the crew", "hand this ticket off", "spawn a worker for KB-XXXX", "kick the crewmate
  off". Mate-tier (runs at cwd=frigate). AIO-specific.
---

# snd-brief — dispatch + supervise a crewmate

The mate never codes tickets itself. It sets up an isolated worktree, launches a crewmate that runs the
**whole** AIO SDLC via the **`snd-sdlc`** skill, and supervises through the **FLEET STATUS** protocol. The crew owns
the pipeline end-to-end **through the draft PR** (kickoff → build → housekeeping → testing → draft PR,
with Jira across it). The captain owns only the human gates after that: Validation (QA), flipping to a
**non-draft** PR, and Merge.

This is the mate's single dispatch skill: *set up → launch → supervise → collect.*

## 1. Set up the worktree

Create an isolated worktree on the feature branch off `RC`, in the project's herdr workspace (the mate
keeps one workspace per project — `herdr workspace list` for the id). Branch: **`KB-XXXXX_descriptor`**
(key + a 1–2 word summary):

```bash
herdr worktree create --workspace <project-ws> --branch KB-XXXXX_descriptor --base RC
# or by path if that workspace isn't open yet:
herdr worktree create --cwd <repo> --branch KB-XXXXX_descriptor --base RC
```

The crew's Planning phase (`snd-kickoff`, via `snd-sdlc`) is **branch-aware** — it picks up this branch
and does the Jira intake (sprint / subtasks / estimate / In Progress) itself; you don't pre-square Jira.

**Provision local skills into the worktree.** A worktree is a fresh checkout, so untracked/**local**
skills (the `snd-*` SDLC set — local until graduated to the team repo) don't propagate. Symlink them in,
skipping tracked team skills already in the checkout (never symlink the whole dir — it clobbers them):

```bash
mkdir -p <worktree>/.claude/skills
for d in <repo>/.claude/skills/*/; do name=$(basename "$d")
  git -C <repo> ls-files --error-unmatch ".claude/skills/$name" >/dev/null 2>&1 && continue
  ln -sfn "$d" "<worktree>/.claude/skills/$name"; done
```

Verify `ls <worktree>/.claude/skills` shows the pipeline: `snd-sdlc`, `snd-kickoff`, `snd-housekeeping`,
`snd-testing`, `snd-pr`, `snd-jira`, `snd-merge-check`. (Once the bundle is graduated to the team repo,
this whole step is unnecessary — the skills are in the checkout.)

## 2. Compose the brief (thin — the SDLC lives in the repo)

The crew runs at `cwd=<worktree>` — an snd_aio checkout — so it auto-loads the repo `CLAUDE.md`, the
`snd-*` skills, and `docs/sdlc.md`. The brief only hands off the task + the launch command + the
protocol. **Don't** inline the SDLC or point at `frigate/docs/...` — the crew can't see frigate.

---
**Task — KB-XXXXX: \<summary\>**  (https://gravitatedxp.atlassian.net/browse/KB-XXXXX)
\<1–3 lines: what to build/fix; note small-bug vs feature.\>

**Run the SDLC.** You're in an isolated worktree on branch `KB-XXXXX_descriptor` (off `RC`) — work only
here; don't touch other worktrees or the main checkout. Drive this ticket with the **`snd-sdlc`** skill in
**`auto` mode**: it walks the whole pipeline — kickoff → build → housekeeping → testing → **draft PR** —
**one phase per turn**, with Jira handled across it, per your `docs/sdlc.md`. You own it **through the
draft PR + Ready for Review**. **Never open a non-draft PR** — that's the captain's call.

**You start in plan mode — design first.** Your very first job is to read the ticket + the relevant code
and **present an implementation plan** (ExitPlanMode). That's the design gate: I review/approve it before
you build. Don't create the branch or move Jira until it's approved — `snd-kickoff` walks you through it.

**Report — one phase per turn, FLEET STATUS.** I'm the **mate**, watching you through herdr, not a human at
a keyboard. Do **one phase, then end your turn** on a FLEET STATUS block (as `snd-sdlc` directs) so herdr
marks you idle and I can read it — I'll continue you into the next phase (or answer a decision):

```
=== FLEET STATUS ===
ticket:   KB-XXXXX
phase:    <phase just finished> → <phase next>
state:    ok | blocked | needs-decision | failed | done
summary:  <one line — what the phase produced, or where you're stuck>
question: <only if blocked/needs-decision — the exact decision I must make>
handoff:  <only if done — branch, PR #, test result>
=== END ===
```
Don't stall on open questions — surface them as `needs-decision` and I'll answer.
---

## 3. Pre-flight, then spawn + send

**a. Does the worktree still have a workspace?** herdr can drop a worktree's workspace on restart while
the git worktree survives (the board keeps its row because it joins on **worktree path**, not workspace
id). If no `herdr workspace list` entry has this worktree as `worktree.checkout_path`, recreate one —
don't spawn into a stale/absent one:

```bash
herdr workspace create --cwd <worktree> --label <ticket-descriptor> --no-focus
# parse result.workspace.workspace_id and result.root_pane.pane_id from the JSON
```

**b. ALWAYS pin the workspace when spawning.** Without `--workspace`, `herdr agent start` splits into the
**focused** pane — i.e. the mate's own tab, not the worktree. Cleanest is to run the agent as the
workspace's root pane:

```bash
herdr pane run <root_pane> "claude --permission-mode plan"   # <root_pane> from (a) — boots in plan mode
# or into an already-open project workspace:
#   herdr agent start KB-XXXXX --workspace <ws> --cwd <worktree> -- claude --permission-mode plan
```

**`--permission-mode plan` is the design rail** — the crew boots read-only and *must* present a plan
(ExitPlanMode) before it can write code, so the design gate (step 4) can't be skipped.

> **Skills load at startup.** If provisioning added a skill *after* the agent was already running, it
> won't see it — restart the pane: `/exit` (Ctrl-C only clears the input; herdr rejects `C-d`), then
> `herdr pane run <pane> "claude --permission-mode plan"` again (a just-provisioned crew is still at
> Planning, so it should reboot into the plan gate).

**c. Deliver the brief.** `herdr agent send` writes literal text but does **not** press Enter — submit
separately. A multi-line brief is easiest from a file (backticks/newlines survive intact):

```bash
herdr agent send <pane> "$(cat /tmp/brief-KB-XXXXX.md)"
herdr pane send-keys <pane> Enter
```

Update the board: the ticket's row → the crew is 🔄 (auto once herdr sees the agent working).

## 4. Supervise + converse (the back-and-forth)

**herdr doesn't push state changes at you — you only learn a crew finished if you're waiting on it.** So
the instant you dispatch (or send a steer), arm the wait in the **background** so the finish re-invokes
you while you stay free for other crews and the captain. Never leave a working crew unwatched.

```bash
# run_in_background: true — you get re-invoked when the crew hits done/idle/blocked
herdr wait agent-status <pane> --status done --timeout 1800000
herdr agent read <pane> --source recent --lines 60            # then read its FLEET STATUS block
```

Act on `state:` — and use `phase:` to move the board's phase track:

- **ok** → a phase boundary. The crew has **stopped and is waiting on you** (one phase per turn). Update
  the board's phase, then **continue it** — `herdr agent send <pane> "continue"` + `herdr pane send-keys
  <pane> Enter` — and re-arm the background wait. *This is what `auto` means: you, the mate, are the
  per-phase continue button, no human needed.* (In a captain-driven/non-auto dispatch, you'd surface the
  boundary and let the captain say continue.)
- **done** → the pipeline reached its terminal: **draft PR open + Ready for Review.** Sanity-check the
  `handoff` (branch, PR #, tests), update the board (**Review**), and **surface to the captain** — the
  remaining gates (Validation/QA → Merge) are theirs. The mate no longer runs the PR/Jira ceremony; the
  crew's `snd-pr` + `snd-jira` did it.
- **needs-decision** → if it's yours, `herdr agent send <pane> "<decision>"` and `wait` again. If it's the
  captain's call, surface it (board **Blocked** + a ping) and hold.
- **blocked** → **first: is it the plan gate?** Read the pane (`herdr agent read <pane>`) — if you see the
  ExitPlanMode prompt (*"Ready to code?" / "Would you like to proceed?"*), handle it per **The plan gate**
  below. Otherwise it's a work block — resolve with a one-line steer if you can, else surface to the
  captain; put the crew's `question:` in the board's Blocked section.
- **failed** → read the evidence, report to the captain, decide retry vs. hand back.

Keep the loop tight — **answer → `herdr agent send` → re-arm the background `herdr wait`**. Short steers
down, FLEET STATUS blocks up. That's the conversation.

### The plan gate (Planning phase)

The crew boots in **plan mode**, so its **first `blocked` is the design gate** — it has produced an
implementation plan and is waiting for approval before it can write code. The plan is persisted to
`~/.claude/plans/*.md` (newest file) and shown in the pane. Read it, then **triage** — this is where you
absorb the trivial and escalate the real design calls:

- **Auto-approve (routine)** — approve it yourself when the plan is single-file-ish, adds **no** new
  model/endpoint/pattern, changes **no** user-facing behavior, and matches existing conventions:

  ```bash
  herdr pane send-keys <pane> Enter      # selects "Yes, and use auto mode" → crew builds autonomously
  ```

  **Record it** so the captain sees what bypassed them — set the board substep to `plan auto-ok`.

- **Escalate (substantive)** — a real design choice, a schema/API change, a new pattern, security/perf
  sensitivity, ambiguous AC, or you're simply unsure → **relay the plan to the captain** (board **Blocked**
  + ping; the `~/.claude/plans/*.md` file is the artifact to share). On the captain's word:
  - **approve** → `herdr pane send-keys <pane> Enter`
  - **redirect** → select *"Tell Claude what to change"* (read the pane for its option number, e.g. `4`):
    `herdr pane send-keys <pane> 4`, then `herdr agent send <pane> "<captain's feedback>"` + `send-keys
    <pane> Enter`.

After approval the crew flips to auto mode and proceeds; from there it's the normal one-phase-per-turn
loop. **When in doubt, escalate** — you're the relay, the captain owns the design call.

## Notes

- If the crew chats at you *without* the block, still read + answer, but nudge it back to the protocol so
  future signals are machine-clear.
- One crewmate per ticket; run several concurrently across worktrees and supervise each.
- The captain's gates are **Validation, non-draft PR, and Merge** — everything up to the draft PR is the
  crew's via the `snd-sdlc` skill.
