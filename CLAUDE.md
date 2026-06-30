# frigate — working notes

> 🚧 **Scratchpad** while we build the orchestrator. We append rough notes here as we
> go so we don't lose context, then clean + reorganize at the end. Don't over-structure
> this yet. (This file auto-loads as project instructions for Claude Code run in `frigate`.)

## Installing skills (Vercel `skills` CLI)

We add skills with the `skills` CLI. **Always pin the agent to Claude Code.** The CLI has
*no persistent agent pin*, so without `-a claude-code` it fans the install out to all 18+
detected agents (creates `.agents/`, `.junie/`, copilot dirs, …) — we only want Claude Code.

Canonical command:

```bash
npx skills add <owner/repo> -s <skill-name> -a claude-code --copy -y
```

- `-a claude-code` — pin to Claude Code only. Slug is `claude-code`; it reads `.claude/skills/`.
- `--copy` — write real, committable files into the repo (not symlinks).
- `-y` — non-interactive. The Socket/Snyk security scan still prints; **read it** before trusting a third-party skill.

Installs land in `.claude/skills/<name>/` and are tracked in `skills-lock.json`.

### Skills in frigate (`.claude/skills/`)

Installed via CLI:
- `skill-creator` — `anthropics/skills` (author / improve skills) — scan: Safe / 0 alerts.
- `herdr` — `ogulcancelik/herdr` (drive herdr from inside it; gated on `HERDR_ENV=1`) — scan: 3 Socket alerts, all capability-based (single markdown file, no scripts).

Authored / brought in:
- `gravi-cli` — umbrella for the `gravi` CLI (auth, instances, config/creds, `query`, `switch`, tokens, paste, watch). Defers burner lifecycle to `gravi-burners`. Body points to `gravi <cmd> --help` for exact flags.
- `gravi-burners` — `gravi burner` lifecycle (start / autosync / recreate / restart / logs / pods / …). Brought in from the global `~/.claude/skills/gravi-burners` copy and **refreshed** for CLI drift: dropped the removed `extend` command, added `restart`. ⚠️ The global copy at `~/.claude/skills/gravi-burners` is now stale (still documents `extend`) — decide whether to refresh/consolidate it.

## Plugins (Claude Code, project-scoped)

Claude Code **plugins** (bundles of skills + agents + commands + hooks + MCP) are a separate system
from the `skills` CLI. We manage frigate's plugins at **project scope** so they live in this repo
(`.claude/settings.json`) rather than user scope — they travel with the workspace.

- Declare a marketplace in the repo: `claude plugin marketplace add <owner/repo> --scope project`
- Enable a plugin in the repo:       `claude plugin install <plugin>@<marketplace> --scope project`
- Remove from user scope:            `claude plugin uninstall <plugin>@<marketplace> --scope user`
- Project scope writes `.claude/settings.json` (`extraKnownMarketplaces` + `enabledPlugins`). **Commit it.** Changes apply on the **next session restart**.

### Plugins enabled here
- `superpowers@superpowers-marketplace` (`obra/superpowers-marketplace`), v6.1.0 — moved user→project scope. Source of the `superpowers:*` skills (brainstorming, writing-skills, TDD, systematic-debugging, …).

## Skill naming & skill-vs-context

**Prefix project-specific skills by product** (the thing whose workflow varies): AIO = `snd-`.
So AIO SDLC skills are `snd-kickoff`, `snd-jira-housekeeping`, `snd-testing`, etc. A future project
gets its own prefix (crossroads → `xr-`/`cr-`). Cross-project skills stay **unprefixed**:
`brainstorming`, `subagent-driven-development`, `gravi-burners`, `gravi-cli`, `herdr`, `skill-creator`.

**Skill vs context:** only specialized, reusable, triggerable knowledge becomes a *skill*
(`gravi-burners`, `backend-i18n`, `snd-jira-housekeeping`). Mechanical/deterministic steps with no
hidden knowledge — squash+push, "draft PR not full", the branch model, the pipeline sequence — live
as **context** in the operating manual, not as skills.

### Existing AIO-specific skills to retrofit to `snd-` later (NOTED, not yet done)
- `fixbugs` → `snd-fixbugs` (bug-subtask fix flow)
- `backend-i18n` → `snd-backend-i18n`
We'll work these in as we go — captured here so we don't forget.

---

_(append notes below as we work)_
