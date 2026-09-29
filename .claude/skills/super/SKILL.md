---
name: super
description: Read Jira tickets, JQL searches, GitHub PRs/reviews/diffs, and ruff format diffs — and write Jira description / acceptance criteria / status / energy points, and create issues — through the `super` CLI, one bash call per read. Use INSTEAD of the Atlassian MCP for every Jira read (it returns Acceptance Criteria and downloads pasted screenshots, which the MCP does not), instead of hand-rolled `gh api`/`jq` for PR review intake, and instead of running `ruff format` in place. Triggers on any ticket key (KB-/GSD-/XR-/COS-), "read the ticket", "what are the AC", "the subtasks", "transition the ticket", "set it to Ready for Testing", "the PR", "the review comments", "the diff", "format check". Comments are the one thing it cannot do — those still go through the Atlassian MCP.
---

# super

`super read <scheme>://<body>` turns a compact URL into agent-ready text on stdout;
`super write <url>` pushes markdown or a transition name back to Jira. It is on PATH.

**Don't memorize routes — ask the tool.** `super read` lists the schemes;
`super read <scheme>://` lists that scheme's routes; `super --help` is the usage.

## Batch — one call, not a loop

`super read <url> <url> ...` runs every read **in parallel** and prints each under a
`==<url>==` header in the order given. Always prefer one batched call.

Unquoted shell brace expansion batches shared prefixes (a plain shell trick, not a super
feature — it only works **unquoted**):

```bash
super read ruff://shared/{valuation_shared/events,shared_util}.py
```

## jira://

| Route | Output |
|---|---|
| `jira://<KEY>` | summary, created/updated, description, **Acceptance Criteria**, all comments, ADF rendered to markdown |
| `jira://<KEY>/children` | subtasks of a story, or the issues of an epic |
| `jira://<KEY>/status` | current status **and the exact available transition names** |
| `jira://<KEY>/desc`, `jira://<KEY>/ac` | that one field, as round-trippable markdown |
| `jira://<KEY>/points` | the Energy Points estimate, or `no Energy Points set` |
| `jira://<PROJECT>/create` | the front-matter template for a new issue (write-only route; see below) |

- **Start every ticket with `super read jira://<KEY> jira://<KEY>/children`.** The description
  is usually one sentence of context; the real requirements live in the AC and the subtasks.
  Reading only the description is the most common way a crew builds the wrong thing.
- A bare number means `KB-`: `jira://50406` == `jira://KB-50406`. Case is normalized.
- **Pasted screenshots are downloaded** to `/tmp/jira-images/<KEY>/<filename>` and referenced
  from the markdown. `Read` them — a QA bug is usually mostly screenshot. That path is shared
  across crews and re-reads skip existing files; never write there yourself.

### Writing

```bash
super read  jira://KB-1/ac > /tmp/…/ac.md      # edit it
super write jira://KB-1/ac < /tmp/…/ac.md
super write jira://KB-1/status "Ready for Testing"
super write jira://KB-1/points 7
super read  jira://KB/create > new.md            # template: summary / type / parent, then the body
super write jira://KB/create < new.md            # KB-52256: created Story under KB-49253
```

- **Read the field first and keep every `[^unrenderable/N]` marker in place.** They stand in
  for content that cannot survive markdown (images, panels); `write` splices the original
  nodes back wherever the markers still appear. Delete one and you delete that content.
- **Read `/status` before writing one.** The transition name must match what that issue
  actually offers, and the list differs by issue type and current state.
- **`create` takes a project, not a ticket** (`jira://KB/create`) and prints the new key. Front matter:
  `summary` (required), `type` (default `Story`), `parent` (optional epic); the description is the
  markdown below the closing `---`. Set the AC afterwards with `/ac`. **Only the captain creates
  tickets** — the route is for carrying out his ask, never for proposing or filing one yourself.
- **Dev / Validate subtasks** — the AIO readiness gate wants exactly two, bare (summary only). The
  issue type is **`Internal Sub-task`** (`Subtask`, `Sub-task` and `Sub-Task` all 400 "Specify a valid
  issue type"), and `parent` is the **story key**, not the epic:
  ```bash
  printf -- '---\nsummary: Dev\ntype: Internal Sub-task\nparent: KB-1\n---\n' | super write jira://KB/create
  printf -- '---\nsummary: Validate\ntype: Internal Sub-task\nparent: KB-1\n---\n' | super write jira://KB/create
  ```
  `create` has no assignee field, so set the assignee afterwards with the Atlassian MCP
  `editJiraIssue` (`{"assignee": {"accountId": "<id>"}}`; `lookupJiraAccountId` finds the id). `Dev`
  goes to whoever did the work; `Validate` goes to the validator the captain names and **never** to the
  dev. Unnamed means leave `Validate` unassigned.
- **Run each `super write` as its own bare command** — no `cd …;`, loop, or `timeout` in front. The
  allow rule `Bash(super write:*)` is a prefix match, so a wrapped call isn't covered by it.
- **Energy Points is the estimate field** — `points` takes a bare number and resolves the
  custom field by name, so nothing is pinned to a `customfield_*` id. Estimates are the
  captain's call; never set one on your own initiative.
- Scratch files go in the session scratchpad, never a shared `/tmp` path — crews collide there.

## jql://

```bash
super read "jql://$(python3 -c 'import urllib.parse,sys;print(urllib.parse.quote(sys.argv[1],safe=""))' '<jql>')"
```

Quote the whole argument; the JQL itself must be URL-encoded. Output is a flat TOON table —
schema declared once, one row per issue, nested objects (assignee, status, parent) already
projected to their display value. **If it ever comes back as raw nested JSON, that's a
regression — flag it, don't work around it.** Default fields are the 8 in the schema line;
`SUPER_JQL_FIELDS='*navigable'` is the firehose escape hatch.

## pr://

Run from inside the repo's worktree — **cwd is how `gh` infers the repo**, there is no `-R`.

| Route | Output |
|---|---|
| `pr://<n>` | state, author, **`head → base`**, review decision, merge state, diff stat, full body |
| `pr://<n>/reviews` | one line per review: reviewer, verdict, timestamp, sha, inline-comment count |
| `pr://<n>/reviews/<i>` | that review's inline comments, each as `file:line` + text |
| `pr://<n>/diff` | raw unified diff |

`head → base` is the load-bearing line on a stacked branch: an AIO PR's base is frequently
another ticket's branch, not `RC`.

## ruff://

`super read ruff://<path>` runs `ruff format <path> --diff`. It **never rewrites the file**.

- Run it over every touched file once a coding pass is done, batched into one call, then
  apply the reported hunks **by hand with Edit**. Running `ruff format` directly rewrites the
  whole file, including code outside your change.
- **Never pass `--unsafe-fixes`.** It has rewritten Beanie queries into something that no
  longer means the same thing.

## What super does NOT do

- **Comments.** Posting or reading a Jira comment thread beyond `jira://<KEY>`'s rendering
  goes through the **Atlassian MCP**. The one-comment-per-ticket policy (test coverage only)
  is unchanged.
- **CI checks.** There is no checks route. Use `gh pr view <n> --json statusCheckRollup` —
  and note that `gh pr checks --json` does not exist on this box (it prints usage and exits
  `0`), and a *running* check has an **empty `conclusion`**, so `.conclusion // .status` in
  jq reports a false green. Require the check count to hold steady too.
- **Non-image attachments.** `frigate/bin/jira-attach <KEY>` still covers those.
