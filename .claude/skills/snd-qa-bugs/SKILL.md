---
name: snd-qa-bugs
description: >-
  Detect and intake QA-filed bug subtasks on in-flight AIO stories, then put the crew on them. Sweeps
  Jira for bug subtasks parented to board tickets, diffs against the ledger, triages each (auto-dispatch
  the clear ones, escalate the unclear to the captain), briefs the existing crewmate in its worktree, and
  drives the Jira status track — bug to In Progress, then Ready for Testing, and the PARENT back to Ready
  for Testing once the last bug clears, or QA is never triggered to re-validate. Use on the reconcile
  heartbeat, at fleet boot, or when asked "any QA bugs?", "check for new bugs", "did QA file anything",
  "put the crew on that bug". Mate-tier (runs at cwd=frigate). AIO-specific.
---

# snd-qa-bugs — QA bug intake

QA files bugs as **Jira bug subtasks on the story the crew is working**. Nothing pushes that at the
mate, so this skill is how the mate finds out and acts: **sweep → diff → triage → dispatch → drive Jira**.

**Why a sweep and not a webhook.** The mate is an intermittent consumer — a webhook fired while no
session is up is lost, silently. Jira is already the durable queue, so a JQL sweep re-derives complete
truth on every run and self-heals across compaction, restart, and overnight. Volume is ~1–2 bugs/day;
detection inside the 20-minute reconcile beats the human it replaces.

## 1. Sweep

Build the parent list from `fleet.json` `in_flight`, then query via the Atlassian MCP
(`searchJiraIssuesUsingJql`, `cloudId: gravitatedxp.atlassian.net`):

```
project = KB AND issuetype = "Bug subtask" AND parent in (<in_flight tickets>)
  AND status not in (Done, Closed, "Ready For Testing")
  AND (assignee = currentUser() OR assignee IS EMPTY) ORDER BY created DESC
```

> **`assignee` is a required clause, not a refinement — a subtask on our story is not automatically
> our work.** Being a bug subtask of a ticket a crew holds says only *where* the defect is, never *who
> owns fixing it*. QA files against the story; the lead then assigns each subtask, sometimes to another
> dev and sometimes keeping it. Sweep without the assignee clause and you intake another person's
> ticket: you dispatch a crew onto it, the crew moves it to In Progress, and the real owner finds their
> subtask claimed and half-built by someone else. **Observed 2026-08-10** — KB-50036 on KB-47631
> (reported by Christine, assigned to Tatum) was dispatched to Charlie and had to be reverted: crew
> stood down, bug back to Open, parent back to Ready for Review.
>
> **Fetch `assignee` in `fields` and check it again before dispatching** — the JQL is the guard, but
> a stale result or a hand-built query shouldn't be the only thing between a crew and someone else's
> work. No assignee match, no dispatch.
>
> **`assignee IS EMPTY` is an escalation, never an auto-dispatch.** An unassigned bug on our story is
> probably ours but nobody has said so. Surface it to the captain — *"KB-xxxxx is unassigned on
> <ticket>, claim it?"* — and dispatch only on his word. Mark it `"triage": "escalated"`.

> **`Ready For Testing` must be excluded, not just `Done`/`Closed`.** A bug at Ready For Testing is
> QA's — the fix shipped and QA hasn't closed it yet. Leave it in the result set and you get a
> **re-intake loop**: step 5 clears the `bugs` array once the parent bounces, so the next sweep sees the
> key missing from the ledger, calls it new, and dispatches the same fixed bug again. If QA rejects the
> fix it moves the bug back to Open/In Progress and the sweep picks it up for real.

`issuetype = "Bug subtask"` carries the type filter — **`Internal Sub-task`** (Dev / FE Dev / Validate,
filed by Automation for Jira) and **`Test`** are the noise and drop out on it. Don't filter by reporter;
QA reporters vary (Christine files, Tatum triages) and the type is the contract. **Assignee is the
ownership filter and reporter is not a substitute for it.**

> **Always pass a narrow `fields`** — `summary,status,parent,assignee,created` — and a bounded `maxResults`. The
> unscoped query returns ~230 KB and blows the token limit outright. **Leave `description` out of the
> sweep**: it's the biggest field and you only need it for a bug you're actually dispatching, so fetch it
> per-bug with `getJiraIssue` at brief time. Note the MCP returns expansions (assignee, project, the
> parent's own status) whatever you ask for, so keep the parent list to real in-flight tickets.

## 2. Diff

Compare the returned keys against each ledger item's `bugs[].key`. A key already present is **not** a
new intake — that's what makes the sweep idempotent and safe to run every 20 minutes. Anything new is
an intake; record it on its parent's item:

```json
"bugs": [
  {"key": "KB-49828", "summary": "<short — this renders on the board>",
   "state": "open", "triage": "auto", "seen": "2026-08-07"}
]
```

`state` walks **`open` → `dispatched` → `fixed`**. Drop the entry once the fix is pushed and the bug
subtask is at Ready for Testing. Also **reconcile the other direction**: a bug the ledger still calls
open that Jira now shows Done/Closed (QA validated it) gets dropped.

## 3. Triage — hybrid (captain, 2026-08-07)

The captain's review stays in the loop only where it earns its keep.

**Auto-dispatch** when *all* hold:
- **the bug is assigned to us** (`assignee = currentUser()`) — re-check the field, don't trust that
  the JQL carried it;
- the parent is an in-flight item, not already merged;
- the crew's pane is alive (`herdr agent list`);
- the bug is plainly a regression in the surface *that crew changed*;
- it touches no hands-off area.

**Escalate to the captain** when *any* hold:
- it is **unassigned** — probably ours, but nobody has said so; ask before claiming it;
- it touches **OrderMovements** — hands-off: no fixes, no follow-up tickets, no neighbour sweeps;
- it spans another ticket's surface, or belongs to a different story;
- it's an AC or design question dressed as a bug;
- it raises a perf or security concern;
- the parent is already Shipped/merged;
- you're unsure.

Escalating sets the item's **one-line** `blocked` flag and pings the captain; the reasoning goes in
`detail`. Mark the entry `"triage": "escalated"`.

The ping is **one glyph line** (manual → *Voice*), naming the bug, the parent, and the actual call —
❓ **Delta · KB-49301** — QA bug KB-49340 is unassigned, claim it? — not "a QA bug needs your triage".
The same line (glyph optional) goes in `blocked`; everything else goes in `detail`.

> **Auto-dispatched bugs do NOT set `blocked`.** They need nothing from the captain — the 🐞 marker and
> the `QA bugs — open` section are how he sees them. Flagging every filing would light the whole board
> red and the feedback section would stop meaning anything.

## 4. Dispatch — prompt the existing crew

**No new worktree.** Crew panes and worktrees survive completion, so the bug goes to the crewmate that
already has the branch and the context:

```bash
herdr agent prompt <pane> "$(cat /tmp/qa-bug-KB-49828.md)"
# then, backgrounded, per snd-brief §4:
herdr agent wait <pane> --until done --timeout 1800000
```

Brief format — mirrors FLEET STATUS so the intake is machine-clear:

```
=== QA BUG ===
bug:      KB-49828  (bug subtask of KB-49254)
reporter: QA
summary:  <one line>
repro:    <steps + expected/actual, from the Jira description>
scope:    fix on your existing branch <branch> — no new worktree, no re-plan
jira:     move KB-49828 to In Progress now; Ready for Testing when the fix is pushed
=== END ===
```

**The crew's drill** (state it in the brief if the crew hasn't done one before):

1. Move the **bug subtask to In Progress**.
2. Reproduce it first — a QA repro that doesn't reproduce is a finding, not a fix; report it.
3. Fix on the **same branch**. Cover it with a test where the bug is testable.
4. **Mini housekeeping** — a *light* pass, not the full `snd-housekeeping` ceremony, but the rules are
   not optional on a bug fix: a light code review of the diff, every applicable repo convention
   enforced (comment/docstring rules included), comment pruning, no ticket refs left in code, lint
   clean.
5. Push (CI).
6. Move the **bug subtask to Ready for Testing** with a comment saying what the fix was — QA closes it
   after re-validating; the crew doesn't close QA's bug.
7. Report FLEET STATUS: `phase: Validation → Validation`, `state: ok`.

**Fallback** when the pane is dead or the workspace was dropped: recreate the workspace on the
**existing** worktree per `snd-brief` §3a and re-brief onto the same branch. Never cut a fresh worktree
for a QA bug.

Then set `bugs[].state = "dispatched"` and re-render (`bin/fleet --write`).

## 5. Drive the Jira status track — this is the part that's easy to drop

Bugs have their own status track, and the **parent's** status is what gates QA:

| What | Status |
|---|---|
| Bug subtask, filed | `Open` |
| Bug subtask, crew working it | `In Progress` |
| Bug subtask, fix pushed | `Ready for Testing` → QA re-validates and closes it |
| **Parent story, while any bug is open** | `In Progress` — QA bounced it back |
| **Parent story, once the LAST bug clears** | **back to `Ready for Testing`** |

**Putting the parent back to Ready for Testing is mandatory and the whole point.** QA is triggered to
re-validate by the *parent's* status, not by the bug subtasks. Leave the parent In Progress after
clearing its bugs and the story silently stalls — the fixes sit there and nobody re-tests them.

So when the last `bugs[]` entry reaches `fixed`: transition the parent to **Ready for Testing**, set the
item's `jira` to match, clear the `bugs` array, and drop the substep back to something honest. The
board's phase stays **Validation** throughout — the ticket never left QA's gate.

Also keep the item's `jira` field honest against real Jira on every sweep. It's hand-maintained, so it
drifts — the parent sitting `In Progress` while the ledger claims `Ready For Testing` is exactly the
staleness this sweep should catch.

## 6. Board

Set on the ledger item; `bin/fleet` does the rest:
- `bugs[]` → a **`🐞 N`** marker on the phase's continuation line, the row floats just under the blocked
  ones, and each bug lists under **`🐞 QA bugs — open`** (with `· dispatched` once the crew has it).
- **A bug does not roll the phase back.** The ticket really *is* at QA's gate, so the phase stays
  Validation; the bug count is what says it's bouncing. Rolling the track backwards on every filing
  would churn it and lose where the ticket actually sits.
- Keep `substep` to one terse token — `QA bounce` is enough; the 🐞 count carries the rest.

## Upgrade path (not built)

Detection is ≤20 min because it rides the reconcile heartbeat. For near-realtime: mint a Jira API token
(there is none on the box today — no token, no `jira`/`acli` CLI, only the interactive-OAuth Atlassian
MCP, which no shell script can use), then a `Monitor` poll loop emits one notification per newly-seen
bug key and re-invokes the mate directly. The same token would let `bin/fleet` read **live** Jira status
instead of the hand-maintained `jira` field.

**Rejected: a real webhook** (Jira Automation → ngrok → local listener). Needs a Jira admin change and a
stable tunnel; free ngrok URLs rotate per restart and a stale webhook fails *silently*; and it still
needs this sweep as a backstop for anything filed while the session is down. Don't re-litigate it.
