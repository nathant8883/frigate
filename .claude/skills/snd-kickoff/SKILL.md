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

## 3. Create the worktree + branch

Branch convention: **`KB-XXX_descriptor`** — the ticket key plus a 1–2 word description drawn from
the summary (e.g. `KB-4821_route-export`). Confirm the descriptor with the captain if it's ambiguous.

Base off **`RC`** (the active integration branch in the `RC → beta → sharedprod` model). Create an
isolated worktree with herdr:

```bash
herdr worktree create --branch KB-XXX_descriptor --base RC
```

If you're not running under herdr, fall back to a sibling git worktree (never branch inside the
primary checkout):

```bash
git -C <repo> worktree add ../.worktrees/KB-XXX_descriptor -b KB-XXX_descriptor RC
```

## 4. Move the ticket to In Progress

Transition the story to **In Progress** (`getTransitionsForJiraIssue` → `transitionJiraIssue`).

## 5. Hand off to development

- **Non-trivial work:** spec + implementation plan via the `brainstorming` skill, then execute via
  `subagent-driven-development`.
- **Small bug/issue:** skip brainstorming; go straight to `subagent-driven-development`.

Report to the captain: the branch name, the worktree path, and the new Jira status — development
starts from there.

---

> Downstream stages (housekeeping, squash+push, testing on burners, draft PR, Jira housekeeping) are
> separate skills / operating-manual steps — see `frigate/docs/snd-aio-sdlc.md`.
