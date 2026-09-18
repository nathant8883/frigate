# Branch stacking under squash-merge

How the fleet runs dependent branches in AIO. Written 2026-08-18, after a five-deep tank-wagon stack
cost most of a day when its bottom two tickets landed.

AIO is **squash-merge only** (`allow_rebase_merge: false`, `allow_merge_commit: false`,
`delete_branch_on_merge: true`), and that is staying. Everything here works *within* that.

## The one thing to understand

When a parent squash-merges, RC gains a single commit holding all of the parent's content. The child
still carries the parent's **individual** commits. Git now sees the same changes twice, from two
unrelated commits, and reports conflicts — often across a dozen files.

**Those conflicts are not real.** Nothing disagrees; the content is already on RC. The instinct to
open the files and resolve them is the trap, and it is actively dangerous: the child's copies of the
shared files are *older* than what just landed, so resolving "take mine" silently reverts work that
is already shipped.

The correct move is never to reconcile. It is to **replay only the child's own commits and discard
the parent's**, by naming the sha the child was cut from as the boundary:

```bash
git rebase --onto supply_and_dispatch_aio/RC <base_sha> <branch> --update-refs
```

### The evidence

Both of these happened on 2026-08-18, in the same repo, against the same conflicting files, hours apart:

| | history shape | result |
|---|---|---|
| **Alpha** (KB-48925) | linear — rebased onto its parent | `--onto` replay, **zero conflicts**, ~10 min |
| **Foxtrot** (KB-48819) | five merge commits pulling the parent in | no usable boundary, full branch rebuild, ~1 h, and its force-push orphaned Bravo |

GitHub reported Alpha's PR as CONFLICTING across 8 files. The replay resolved all 8 by not touching
them. The only variable between the two cases was history shape.

## Rules

**1. Never merge a parent into a child. Rebase onto it.** This is the rule that cost us the day.
Merging is the lower-friction move in the moment and it destroys the boundary permanently — every
future restack of that branch is a rebuild.

**2. Record the boundary at dispatch.** `fleet.json` carries `parent` and `base_sha` per crew.
`base_sha` is the exact sha the branch was cut from. Guessing it later is the archaeology that made
Foxtrot expensive; recording it costs nothing.

**3. Cap stack depth at two, and check the dependency is real.** Cost grows quadratically with depth
— every landing forces a restack of everything above. The 2026-08-18 chain was five deep
(KB-48310 → KB-49849 → KB-48925 → KB-48819 → KB-49652). KB-49652 sat *under* KB-48819 when it only
needed KB-48925's compartment model; beside it rather than below it would have halved the cascade.

**4. Order the stack by merge-readiness, not dependency convenience.** Whatever is closest to
shipping goes lowest. A nearly-done ticket parked under a week-away one pays the restack tax
repeatedly for nothing.

**5. A parent announces before it rewrites history.** `--update-refs` handles the local side, but the
children still need force-pushing, and the mate needs to sequence them. A child discovering its base
has vanished is the failure to design out.

**6. No backup branches.** Push and use `--force-with-lease`; `reflog`/`ORIG_HEAD` hold the
pre-rebase tip. See the manual's *No safety-net branches*.

## Tooling

**`bin/restack <crew> [--dry-run] [--onto <ref>]`** — runs the replay above using the ledger's
recorded boundary, so nobody has to find it. It picks the new base automatically (the parent branch
if it still exists on the remote; RC if it doesn't, since a vanished parent branch means it was
squash-merged and deleted), refuses on a dirty tree, updates the ledger afterwards, and **refuses
outright if the branch range contains merge commits** — that being the condition that makes the
boundary meaningless. Always `--dry-run` first; it prints the exact commits it will replay.

**`rebase.updateRefs = true`** (set globally 2026-08-18) — during a rebase, git moves any local
branch ref pointing at a replayed commit instead of stranding it. This is the direct fix for the
Bravo failure: Foxtrot rewrote its branch and every branch above it silently pointed at commits that
no longer existed.

It works here **only because all of a repo's herdr worktrees share one clone** — every linked
worktree's `repo_key` is the same `projects/<repo>/.git`, so a sibling crew's branch is a local ref
in the same object store. It is a **local** setting; nothing changes on GitHub, and each child still
needs its own `--force-with-lease` push to update its PR.

**`rerere`** — the same files (`best_buy/api.py`, `tank_wagons/*`, `docs/tank_wagons.md`) needed the
same resolutions for both Alpha and Foxtrot. rerere records a resolution once and replays it at every
level of the stack. **Not yet enabled** — deliberately held while crews were mid-rebase, since it
changes conflict behaviour underneath them. Turn on with `git config --global rerere.enabled true`.

## What was considered and rejected

**A long-lived `tw-integration` branch** collecting all tank-wagon work for one merge to RC. It would
genuinely kill the conflicts, but under squash-merge it produces one enormous squash at the end, and
it breaks per-story QA — Validation runs per ticket, so stories have to land individually.

**Enabling `allow_rebase_merge`** on the repo. This is the only change that removes the failure mode
rather than managing it, since it preserves commit identity and lets children fast-forward. Ruled out
by the captain 2026-08-18: squash-merge stays.

## The failure signatures

- **"CONFLICTING" on a child right after a parent merged** → duplicate history. `bin/restack`, do not
  open the files.
- **A child reading hundreds of commits ahead of its own base** → its base was rewritten under it.
  Restack onto the parent's new tip.
- **A restack that conflicts anyway** → those conflicts are real. Resolve them properly.
- **`git cherry` / `git branch --merged` saying a landed branch is unmerged** → expected under
  squash-merge; commit counts prove nothing. The PR state is the signal.
