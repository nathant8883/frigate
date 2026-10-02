---
name: super
description: Read GitHub PRs, reviews and diffs, and ruff format diffs, through the `super` CLI, one bash call per read. Use instead of hand-rolled `gh api`/`jq` for PR review intake, and instead of running `ruff format` in place. Triggers on "the PR", "the review comments", "the diff", "format check". Jira is NOT here any more — use the gravi-jira skill (`gravi-axi jira`) for every ticket read, write and comment; this skill keeps the fleet's Jira rules that apply on top of it.
---

# super

`super read <scheme>://<body>` turns a compact URL into agent-ready text on stdout. It is on
PATH. Use it for `pr://` and `ruff://`; Jira is the gravi-jira skill (see below).

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

## Jira → the gravi-jira skill

Jira moved to **`gravi-axi jira`** (the gravi-jira skill). It acts as the captain through mom's
Atlassian OAuth link, so every edit and comment carries his name. `super jira://` and `jql://`
still work but are retired here; don't use them.

| was | now |
|---|---|
| `super read jira://KB-1 jira://KB-1/children` | `gravi-axi jira read KB-1/all` |
| `super read jira://KB-1/<route>` | `gravi-axi jira read KB-1/<route>` |
| `super read "jql://<url-encoded>"` | `gravi-axi jira jql '<plain JQL>'` |
| `super write jira://KB-1/<field> < f` | `gravi-axi jira write KB-1/<field> < f` |
| `super write jira://KB/create < f` | `gravi-axi jira create KB < f` |
| Atlassian MCP comment | `gravi-axi jira comment KB-1 < f` |

Fleet rules on top of the gravi-jira skill:

- **Only the captain creates tickets.** `create` carries out his ask; never use it to propose or
  file one yourself.
- **Dev / Validate subtasks:** exactly two, bare, type `Internal Sub-task`, parent = the story.
  `Dev` goes to whoever did the work. `Validate` goes to the validator the captain names and
  **never** to the dev; if no one is named, leave it unassigned (`gravi-axi jira write <KEY>/assignee <email>`).
- **Energy Points are the captain's call.** Never set one on your own initiative.
- **One comment per ticket, test coverage only.** `gravi-axi jira comment` posts as the captain,
  so nothing else goes out under his name without his say-so.
- **Run each `gravi-axi jira write`/`create`/`comment` as its own bare command**, with no `cd …;`,
  loop or `timeout` in front, so a prefix permission rule can match it.
- Scratch files go in the session scratchpad, never a shared `/tmp` path (crews collide there),
  and never into `/tmp/jira-images`.

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

- **Jira.** See "Jira → the gravi-jira skill" above.
- **CI checks.** There is no checks route. Use `gh pr view <n> --json statusCheckRollup` —
  and note that `gh pr checks --json` does not exist on this box (it prints usage and exits
  `0`), and a *running* check has an **empty `conclusion`**, so `.conclusion // .status` in
  jq reports a false green. Require the check count to hold steady too.
- **Non-image attachments.** `frigate/bin/jira-attach <KEY>` still covers those.
