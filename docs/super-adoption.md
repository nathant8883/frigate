# Adopting `super` into the fleet

`super` (westaylor-gravitate/super) is Wes's WIP Rust CLI. Half of it is a GNOME `win+space`
overlay daemon we will never run; the other half is `super read <scheme>://<body>` /
`super write <url>`, a sidecar that turns a compact URL into agent-ready text on stdout. That
half replaces the Atlassian MCP for reads, replaces raw `gh` calls for PR review intake, and
gives crews a Jira **write** path for desc / AC / status.

Reviewed at commit `7d1bbca` (2026-09-18). One commit, no CI, WIP — pin to a smoke-tested sha,
don't track his default branch blindly.

## What we get

| URL | Output |
|---|---|
| `jira://<KEY>` | summary, created/updated, description, **AC (`customfield_10110`)**, all comments, ADF → markdown, **pasted screenshots downloaded** to `/tmp/jira-images/<KEY>/` |
| `jira://<KEY>/children` | subtasks of a story / issues of an epic |
| `jira://<KEY>/status` | current status **+ available transition names** |
| `jira://<KEY>/desc`, `/ac` | that field alone, as round-trippable markdown |
| `jql://<urlencoded>` | flat TOON table, one row per issue, nested objects projected to display values, paginated |
| `pr://<n>` | overview — state, author, **`head → base`**, review decision, merge state, diff stat, body |
| `pr://<n>/reviews`, `/reviews/<i>`, `/diff` | review list; one review's inline comments as `file:line` + text; raw unified diff |
| `ruff://<path>` | `ruff format --diff`, read-only |

- `super write jira://<KEY>/{desc,ac} < markdown` — read the route first, keep the
  `[^unrenderable/N]` markers, and the original nodes (images, panels) splice back on write.
- `super write jira://<KEY>/status "Ready for Testing"` — transition by name.
- `super read a b c` runs in parallel, prints under `==<url>==` headers in the order given.
  Unquoted shell brace expansion batches shared prefixes: `super read ruff://shared/{a,b}.py`.
- Self-documenting: `super read` lists schemes, `super read <scheme>://` lists that scheme's routes.

## Prerequisites — state on this box (checked 2026-09-18)

| Thing | State |
|---|---|
| cargo / rustc | 1.97.1 ✅ |
| `gh` | 2.45.0 ✅ (super uses `--json` on `pr view` + `gh api`, both fine on 2.45) |
| `ruff` | on PATH ✅ |
| Jira creds | `JIRA_EMAIL` + `JIRA_API_TOKEN` already exported from `~/.zshrc` ✅ — super wants them at `~/.config/semdiff.json` as `{"email","api_token"}` |
| Tenant | super hardcodes `gravitatedxp.atlassian.net` — same tenant `bin/jira-attach` uses ✅ |
| cairo / xcb dev headers | **not installed** ❌ — `cairo-rs` + `x11rb` are unconditional deps, so a stock `cargo build` fails |
| `~/.claude/skills` | **exists** (contains a Claude-managed `synced/` bucket) — user-scope skills now reach every crew |

## Phase 0 — build and install

1. Clone to `projects/super`, remote `origin` = `westaylor-gravitate/super`, work on branch `frigate`.
2. **Feature-gate the overlay** so the read/write half builds with no system libraries:
   - `Cargo.toml`: `x11rb` / `cairo-rs` → `optional = true`; `[features] overlay = ["dep:x11rb", "dep:cairo-rs"]`, default `[]`.
   - `#[cfg(feature = "overlay")]` on `config`, `dispatch`, `gsettings`, `install`, `overlay`,
     `proto`, `serve`, `socket` — and on the `serve` / `emit` / `install` / `uninstall` arms in
     `main.rs`. `read/jira.rs` calls `install::home_dir`, so that one helper moves out of the
     gated module (or gets its own tiny ungated module).
   - Build: `cargo build --release --no-default-features`.
   - *Alternative, not recommended:* `sudo apt install libcairo2-dev libxcb1-dev libx11-dev …`
     — installs GUI dev deps for a daemon we never start, and leaves us unable to build on any
     box that lacks them.
3. Symlink `target/release/super` → `~/.local/bin/super`, matching `burner-live` / `gravi`.
4. Write `~/.config/semdiff.json` (chmod 600) from the existing env vars.
5. **Smoke tests, in this order** — #1 is the likely blocker:
   - `super read jira://KB-50563` — super talks to `api.atlassian.com/ex/jira/{cloud_id}` with
     **Basic** auth. `bin/jira-attach`'s own docstring records a 403 doing exactly that, which is
     why it falls back to the site host. If this 403s, the fix is a token scope, or pointing
     `SUPER_JIRA_API_BASE` at `https://gravitatedxp.atlassian.net/rest/api/3`.
   - A ticket with a **pasted screenshot in a comment** — the only path exercising media
     resolution against `renderedBody`; check `/tmp/jira-images/<KEY>/`.
   - `super read pr://<n> pr://<n>/reviews` from an AIO worktree (cwd is how `gh` infers the repo).
   - `super read "jql://<encoded>"` — confirm the TOON table, not raw nested JSON.
   - `super write jira://<throwaway>/status "<name>"` before trusting it on a live story.

## Phase 1 — reach every crew

- Authoritative skill in `frigate/.claude/skills/super/`, symlinked to `~/.claude/skills/super`
  — version-controlled here, loaded from any cwd, same pattern as `gravi-cli` / `gravi-burners`.
- `snd-brief` also symlinks it into each worktree at dispatch (belt and braces; crews booted
  before this have neither, so check `ls <worktree>/.claude/skills`).
- Update the **Toolbelt** table in `frigate/CLAUDE.md`: Jira/PR reads go through `super`; the
  Atlassian MCP is demoted to **comments only**.

## Phase 2 — rewire by brief, not by editing the project skills (captain, 2026-09-18)

**The `snd-*` SDLC skills are not edited.** They live in each worktree as untracked copies with no
canonical upstream (five live crews, five identical directories, nothing owning them), so a tool
change patched into them means five edits under five running crews, drifting from the moment one is
touched and invisible to the next worktree created.

The instruction lives at **frigate level** instead:

| Where | What it says |
|---|---|
| `snd-brief` §2 (the dispatch brief) | intake is `super read jira://<KEY> jira://<KEY>/children`; status / desc / AC writes, `pr://` review intake and `ruff://` format checks all go through `super`; **"this overrides what the `snd-*` skills say"**; comments stay on the MCP |
| `CLAUDE.md` Toolbelt | `super` is the Jira/PR read path for any crew, from any cwd |
| `CLAUDE.md` → *Fleet tooling is briefed at dispatch* | the general rule this is an instance of |
| the `super` skill | the routes, the batching, the `[^unrenderable/N]` rule, what it can't do |

A project skill still naming `getJiraIssue` or hand-rolled `gh api` is **not a bug to fix in the repo** —
the brief is the authority, and it says so out loud because the crew reads both.

## Phase 3 — reopened for one route (captain, 2026-09-22)

The as-is posture below held for four days and then paid for itself the first time it cost
something: setting an Energy Points estimate was the last ticket field with no `super` route, so
it fell back to the Atlassian MCP — which on this box timed out twice on `editJiraIssue` before
the write landed. The captain's call was to build the route rather than keep the exception.

**`jira://<KEY>/points` — read and write the Energy Points estimate.**

```bash
super read  jira://KB-50387/points      # KB-50387: 7   (or "no Energy Points set")
super write jira://KB-50387/points 7    # KB-50387: set Energy Points to 7
```

- **The field id is resolved by name, not pinned.** Energy Points is a site-local custom field, so
  unlike `description` there is no portable `customfield_*` to hardcode the way
  `ACCEPTANCE_CRITERIA_FIELD` (`customfield_10110`) is. `points_field_id` looks it up against
  `/field`. That costs one extra GET, and it means the route needs no edit if the id ever moves.
- **Jira stores the value as a number**, so a whole estimate comes back as `7.0`; the read prints
  it the way it was written.
- **Estimates are owner-set.** The route exists so the mate can carry out the captain's call, not
  so a crew can estimate its own ticket. Never set one unasked.

### How a `super` change gets made — the shape to reuse

Upstream is Wes's repo and our `frigate` branch is **uncommitted build plumbing**, not a fork:
`frigate` points at `origin/master` exactly, and the plumbing (the feature-gated overlay,
`src/home.rs`) lives only in the working tree. So a contribution cannot just be committed where it
sits — the plumbing would ride along.

1. Cut a worktree off **`origin/master`**, not off the working tree.
2. Apply only the feature's files as a patch. **Watch the one-line trap:** our tree has
   `use crate::home::home_dir;` where upstream has `use crate::install::home_dir;` — that line is
   plumbing and must not enter the PR.
3. **The suite cannot run on clean `master` on this box** — `cairo-sys-rs` needs cairo/X11 headers
   that aren't installed, which is the whole reason the plumbing exists. Layer the plumbing patch
   on temporarily to get a build, run the suite, then `git checkout` those files back out so the
   commit is feature-only. Verify the final `git diff --stat` before committing.
4. Draft PR to `master`. Non-draft stays the captain's gate.

First one through this path: **PR #3**, branch `jira-points`.

## Phase 3 (original) — not taken (captain, 2026-09-18)

`super` is adopted **as-is**. The two gap routes (`pr://<n>/checks`, `jira://<KEY>/comment`)
are **not** being built, and we are not forking for features — only the build plumbing in
Phase 0. Still true of *these two*; `points` above is the one route since taken. Consequences,
stated so nobody re-derives them:

- **Comments stay on the Atlassian MCP.** `/post-tests` coverage comments and the E2E yes/no
  reply go through the MCP; `super` never posts a comment. The one-comment-per-ticket policy
  is unchanged.
- **CI checks stay on `gh`.** `super` has no checks route, so the two traps remain live:
  gh 2.45.0 has no `--json` on `pr checks` (prints usage, exits `0`), and a *running* check has
  an empty `conclusion`, so `.conclusion // .status` in jq reports a false green. Use
  `gh pr view <n> --json statusCheckRollup` and require the check count to hold steady.

## Notes and risks

- `/tmp/jira-images/<KEY>/` is hardcoded and shared across crews. Reads are idempotent
  (existing files are skipped), so concurrent crews are fine — but nothing else may write there.
- `jql://` defaults to 8 fields; `SUPER_JQL_FIELDS='*navigable'` is the firehose escape hatch.
- `super --help` is agent-facing and deliberately omits the daemon commands, so a crew can't be
  steered into `serve` / `install`.
- Overlaps we keep for now: `bin/jira-attach` (non-image attachments), the Atlassian MCP (comments).
