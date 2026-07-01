---
name: snd-kickoff
description: >-
  Kick off a supply_and_dispatch_aio (AIO) KB ticket for development — verify the story is ready in
  Jira (Dev & Validate subtasks, in the current sprint, has an energy-point estimate), create an
  isolated worktree on a correctly-named feature branch (`KB-XXX_descriptor`), move the ticket to In
  Progress, then hand off to spec/build. Use whenever starting work on an AIO KB ticket — "start
  KB-1234", "kick off this ticket", "begin work on <AIO story>", "set up a worktree for KB-XXXX",
  "pick up KB-XXXX" — even if the skill isn't named. AIO-specific; other products have their own kickoff.
---

# snd-kickoff — start an AIO KB ticket

Prepares a `supply_and_dispatch_aio` KB ticket for development: confirms it's ready in Jira, creates
an isolated worktree on a correctly-named feature branch, sets the ticket In Progress, and hands off
to development. AIO only — the branch model and Jira rules below are AIO's.

Full SDLC context: `frigate/docs/snd-aio-sdlc.md`. Jira: `gravitatedxp.atlassian.net` (Atlassian MCP
tools). Repo/remote: `supply_and_dispatch_aio`.

## Tooling

Drive Jira through the **Atlassian (Jira) MCP** — it's available in this workspace; use it rather
than the Jira web UI or asking the captain to do Jira steps by hand. The tools are deferred, so load
them with ToolSearch as needed. Key ones: `getJiraIssue`, `searchJiraIssuesUsingJql`,
`getTransitionsForJiraIssue`, `transitionJiraIssue`, `createJiraIssue` (subtasks), `editJiraIssue`
(sprint / estimate). Site `gravitatedxp.atlassian.net`, project `KB`; get the cloudId via
`getAccessibleAtlassianResources` or pass the site URL as `cloudId`.

## 1. Resolve the ticket

Fetch the KB ticket with `getJiraIssue` (fields: `summary, issuetype, status, parent, subtasks,
sprint`, and the story-point/energy-point field).

- If it's a **story**, that's the unit of work.
- If it's a **subtask**, work its parent story (a story carries the branch).

## 2. Note Jira readiness (the gate is at PR, not here)

The story needs all three **before it can be PR'd** — not before work starts. At kickoff, just
surface any gaps as a heads-up so they get resolved during dev; **don't block kickoff** on them.
(Never change Jira silently — sprint membership and estimates are the captain's call.)

- **≥2 subtasks — Dev & Validate**
- **In the current sprint**
- **Energy-point estimate**

The gate that actually enforces these before a PR lives in `snd-jira-housekeeping`
(see `frigate/docs/snd-aio-sdlc.md`).

## 3. Create the worktree + provision it

Branch convention: **`KB-XXX_descriptor`** — the ticket key plus a 1–2 word description drawn from
the summary (e.g. `KB-4821_route-export`). Confirm the descriptor with the captain if it's ambiguous.

Base off **`RC`**. Create the worktree **in the project's herdr workspace** so it shows up as a tab
there (the mate keeps one workspace per project — `herdr workspace list` to find the id):

```bash
# preferred — attach to the project workspace
herdr worktree create --workspace <project-workspace-id> --branch KB-XXX_descriptor --base RC
# or by repo path if that workspace isn't open yet
herdr worktree create --cwd <repo> --branch KB-XXX_descriptor --base RC
```

**Provision local skills into the worktree.** The crewmate runs with `cwd = <worktree>`, a fresh
checkout — so untracked/local skills in the main checkout don't propagate into it. Symlink the repo's
**local-only** skills in, skipping tracked team skills (e.g. `backend_testing`) that are already in the
checkout — never symlink the whole `.claude/skills` dir or you'll clobber the tracked ones:

```bash
mkdir -p <worktree>/.claude/skills
for d in <repo>/.claude/skills/*/; do
  name=$(basename "$d")
  git -C <repo> ls-files --error-unmatch ".claude/skills/$name" >/dev/null 2>&1 && continue  # skip tracked
  ln -sfn "$d" "<worktree>/.claude/skills/$name"
done
```

(One-time per project: `**/.claude/skills/` is in the repo's `.git/info/exclude` — local, shared across
worktrees via the common git dir — so our local skills and these symlinks never show in `git status` or
get committed. Share one later with `git add -f .claude/skills/<name>`. Full model:
`frigate/docs/snd-aio-sdlc.md` → **Orchestration**.)

If you're not running under herdr, fall back to a sibling git worktree (never branch inside the
primary checkout), then provision it the same way:

```bash
git -C <repo> worktree add ../.worktrees/KB-XXX_descriptor -b KB-XXX_descriptor RC
```

## 4. Move the ticket to In Progress

Transition the story to **In Progress** (`getTransitionsForJiraIssue` → `transitionJiraIssue`).

## 5. Hand off to development → `snd-brief`

Kickoff prepared the worktree; now **dispatch a crewmate** to do the actual work — don't code it
yourself. Use **`snd-brief`** to compose a self-contained brief and supervise the crew via the FLEET
STATUS protocol. The crew (in the worktree) does the spec/build (`brainstorming` →
`subagent-driven-development`), `housekeeping`, and tests; the mate owns the back-half ceremony.

Report to the captain: the branch name, the worktree path, and the new Jira status — then `snd-brief`
takes it from there.

---

> Downstream stages (housekeeping, squash+push, testing on burners, draft PR, Jira housekeeping) are
> separate skills / operating-manual steps — see `frigate/docs/snd-aio-sdlc.md`.
