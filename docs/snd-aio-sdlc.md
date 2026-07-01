# supply_and_dispatch_aio — SDLC & operating manual

> AIO is the core product. Other projects follow a **similar but not identical** process.
> This is both the **spec** for AIO's lifecycle and the **per-ticket playbook** a crewmate follows —
> the backbone the frigate orchestrator drives. Captured 2026-06-30. **[TBD]** = to define as we build.

## Branch model

- `RC` — Release Candidate
- `beta` — prod for canary releases
- `sharedprod` — prod for all remaining clients

Promotion flows `RC → beta → sharedprod`. **Feature work branches off `RC`.**

## Per-ticket playbook

One KB ticket flows through these stages. `→` marks the skill/tool that owns the stage.

1. **Kick off** → `snd-kickoff`
   Jira readiness heads-up → herdr worktree on `KB-XXX_descriptor` (off `RC`, in the project's
   workspace) → provision crew skills into the worktree → ticket to **In Progress**. (See
   **Orchestration** below.)

2. **Spec + build** — the mate dispatches a crewmate via `snd-brief` (self-contained brief + FLEET
   STATUS report-back protocol); the crew, in the worktree, does:
   - Non-trivial work: `brainstorming` (spec + implementation plan) → `subagent-driven-development` (execute).
   - Small bug/issue: skip brainstorming; go straight to `subagent-driven-development`.
   - Wrap new user-facing strings with `tr()` → `backend-i18n`.
   - Fixing bug subtasks on an existing story branch → `fixbugs`.

3. **Housekeeping** → `housekeeping` (snd_aio skill): simplify → code-review → BE/FE best-practice compliance table (Pass/Fail/N/A) → auto-fix fails.

4. **Squash + push** *(mechanical — context, no skill)*
   - Squash the feature work into a tight set of commits so the initial dev reviews in one pass.
   - ff-push the feature branch to `supply_and_dispatch_aio` — never force a shared branch.
   - The push triggers CI to build the per-SHA image (required before a burner can boot the branch). Watch it:
     ```bash
     gh run list --branch <branch> --limit 8
     gh run watch <run-id> --exit-status --interval 20
     ```

5. **Testing** — "hourglass": heavy unit + E2E.
   - Unit: `uv run pytest` (backend).
   - E2E + manual verification against a **burner** (ephemeral env booted off the branch image; loads a dataset; runs E2E suites) → `gravi-burners`. **[E2E / snd_tests run specifics TBD]**

6. **Draft PR** *(mechanical + rule — context, no skill)*
   - **First, the pre-PR Jira gate** → `snd-jira-housekeeping`: ≥2 Dev/Validate subtasks, in the current sprint, energy-point estimate. If a gap is unresolved, **stop — don't open the PR.**
   - Document **AC drift → Dev Implementation** section + post the **test-coverage comment** (gherkin + unit tests) → `snd-jira-housekeeping`.
   - Open a **draft** PR only: `gh pr create --draft --base RC --body "<filled template>"`. **Never a full/ready PR without the captain's approval** (full PRs create review noise for the team). Use snd_aio's own convention: the template at `.github/pull_request_template.md` (What / Why / Test plan / Files / Ticket) and the repo `CLAUDE.md` § Pull Requests — fill the body (never blank), pass `--body` not `--fill`, terse/factual tone (cf. #960/#958/#957). A crewmate in the worktree already has both in context (it auto-loads snd_aio's `CLAUDE.md`); the mate, at frigate, reads them from the repo when it opens the PR.
   - Set the story to **Ready for Review** → `snd-jira-housekeeping`.

7. **Ready for Testing** → `snd-jira-housekeeping`
   Once dev + E2E are done and it's ready for third-party manual validation, move the story to **Ready for Testing**.

## Orchestration — herdr workspaces & skill placement

frigate drives the SDLC through **herdr** (running daemon, session `default`). The hierarchy:

```
session (herdr daemon)
└── workspace        one per project   — herdr workspace create --cwd projects/<repo> --label <repo>
    └── tab          one per worktree  — herdr worktree create --workspace <id> --branch KB-XXX_descriptor --base RC
        └── pane → agent  the crewmate — herdr agent start claude --workspace <id> --cwd <worktree> -- claude
```

- The **mate** (orchestrator) runs at `cwd = frigate`. It keeps one workspace per active project,
  opens a worktree-tab per ticket, starts a crewmate agent in each, and watches them with
  `herdr agent list` / `herdr wait agent-status <pane> --status done`.
- A **crewmate** runs at `cwd = <worktree>` — a separate git checkout (herdr puts worktrees outside
  the repo). That cwd is the constraint everything below hinges on.

### Mate skills vs crew skills

Skills resolve from the **running agent's cwd** (`<cwd>/.claude/skills` + `~/.claude/skills`). Because
the crewmate's cwd is the worktree, **frigate's `.claude/skills` are invisible to it.** So skills split
by who runs them:

- **Mate skills** — run at `cwd=frigate`: `snd-kickoff`, `snd-jira-housekeeping` (intake + PR/Jira
  ceremony that bracket the work). **Home: `frigate/.claude/skills/`.**
- **Crew skills** — run in the worktree: `brainstorming`, `subagent-driven-development`,
  `(snd-)fixbugs`, `(snd-)backend-i18n`, code-review/simplify, `gravi-burners`. **Home: the project
  repo, local scope (below).**

### Crew-skill placement: project-repo-local, provisioned, graduation-ready

Crew skills live in the **project repo** at `<project>/.claude/skills/`, kept **local** (never pushed,
team unaffected) via the repo's `.git/info/exclude` entry `**/.claude/skills/` — it ignores *untracked*
skill dirs, so already-tracked team skills (e.g. `backend_testing`, the frontend's `grids`/`i18n`) are
unaffected (gitignore never untracks committed files). `info/exclude` lives in the shared common git
dir, so the one entry covers every worktree.

Caveat git forces: a worktree is a fresh checkout, so **untracked files don't propagate into it.** The
mate therefore **provisions** at kickoff — after `worktree create`, symlink each **local-only** skill
into the worktree (skip tracked skills — they're already in the checkout, and symlinking the whole dir
would clobber them):

```bash
mkdir -p <worktree>/.claude/skills
for d in <repo>/.claude/skills/*/; do name=$(basename "$d")
  git -C <repo> ls-files --error-unmatch ".claude/skills/$name" >/dev/null 2>&1 && continue
  ln -sfn "$d" "<worktree>/.claude/skills/$name"; done
```

**Graduation:** to share one skill, `git add -f .claude/skills/<name>` (force past the local ignore) and
commit to `RC` — it becomes shared and worktrees inherit it on checkout; drop that skill's symlink.
Don't delete the `**/.claude/skills/` exclude line itself — that would expose *every* local skill.

**superpowers rider:** `brainstorming` + `subagent-driven-development` come from the superpowers
*plugin*, not loose skills. So crewmates get them, enable superpowers at **user scope**
(`claude plugin install superpowers@superpowers-marketplace --scope user`) — a public plugin, nothing
secret. frigate's project-scope enable stays for the mate.

> Per-project scaffolding (the `info/exclude` entry + the first crew skills) lands when a project is
> brought in / first kicked off. `snd_aio` is now moved in at
> `frigate/projects/supply_and_dispatch_aio` (2026-06-30) — the scaffolding is the next step, not yet applied.

## Jira rules

- Every story: **≥2 subtasks — Dev & Validate**.
- Must be in the **current sprint** before it can be PR'd.
- Must have an **energy-point** estimate.
- Statuses: **In Progress** (work started) → **Ready for Review** (PR open) → **Ready for Testing** (dev complete, e2e done; ready for third-party manual validation).
- **AC drift:** acceptance criteria often shift during dev — document adjustments in a **Dev Implementation** section on the ticket (reflect what was *actually* delivered; leave the original AC intact).
- **Test coverage:** document in a ticket comment — the gherkin from `snd_tests` (if applicable) + the unit tests added.
- Jira: `gravitatedxp.atlassian.net`, project `KB`, via the Atlassian MCP.

## Stage → skill / tool map

| Stage | Skill / tool | Status |
|---|---|---|
| 1 Kick off (Jira intake + worktree/branch + In Progress) | `snd-kickoff` | ✅ built |
| 2 Dispatch (brief + supervise crew) | `snd-brief` (mate↔crew FLEET STATUS protocol) | ✅ built |
| 2a Spec / plan | `brainstorming` | have |
| 2b Execute | `subagent-driven-development` | have |
| 3 Housekeeping | `housekeeping` (snd_aio): simplify → review → BE/FE compliance table + autofix | ✅ built |
| 4 Squash + push + CI | **context** (git + `gh run watch`) | ✅ in this doc |
| 5 Unit / E2E testing | `uv run pytest`, `snd_tests`, `gravi-burners` | partial, **TBD** |
| 6 Draft PR | **context** (`gh pr create --draft --base RC` + repo template) | ✅ resolved — snd_aio's `.github/pull_request_template.md` + `CLAUDE.md` § Pull Requests |
| Jira hygiene (gate, statuses, AC-drift, coverage) | `snd-jira-housekeeping` | ✅ built |
| Bug fixes on a story | `fixbugs` (→ `snd-fixbugs`) | have |
| i18n strings | `backend-i18n` (→ `snd-backend-i18n`) | have |

## Naming & skill-vs-context

New AIO skills use the **`snd-`** prefix (product-based, so future projects namespace cleanly):
`snd-kickoff`, `snd-jira-housekeeping`, `snd-testing`, … Cross-project skills stay unprefixed
(`brainstorming`, `subagent-driven-development`, `gravi-burners`).

Only specialized, reusable, triggerable knowledge becomes a **skill**. Mechanical/deterministic
steps — squash+push, "draft PR not full", the branch model, the pipeline sequence — are **context**
(captured in this manual), not skills.

Existing AIO-specific skills to retrofit to `snd-` later (noted, not yet done): `fixbugs` →
`snd-fixbugs`, `backend-i18n` → `snd-backend-i18n`.

## Open definitions (nail down as we build)

1. **Testing** specifics — the hourglass details; how `snd_tests` E2E suites run against a burner. (The repo's `backend_testing` skill already covers unit-test best practices.)
2. **Jira** — current-sprint detection + energy-point estimation (manual vs assisted). (Transition/field names are resolved at runtime via `getTransitionsForJiraIssue`.)

*Resolved:* feature-branch base = `RC`; worktree mechanism = herdr `worktree create`; **PR template** = snd_aio's own `.github/pull_request_template.md` + `CLAUDE.md` § Pull Requests; **§4.1 housekeeping** = the `housekeeping` skill in snd_aio (simplify → code-review → BE/FE compliance table + auto-fix).

## Status

- ✅ **Built:** `snd-kickoff`, `snd-brief`, `snd-jira-housekeeping` (mate-tier); `housekeeping` (snd_aio, stage 3); this operating manual + the mate layer (dispatch loop, fleet board).
- ▶ **Next:** **testing** specifics + **Jira** sprint/points detection. (§4.1 housekeeping ✅ and PR template ✅ resolved.)
- ⏭ **Later:** the **orchestration layer** — the first mate dispatching a crewmate per ticket and supervising it through this playbook (herdr-driven).

## Where this lives

A frigate reference doc for now (the orchestrator points crewmates at it). `snd_aio` now lives at `frigate/projects/supply_and_dispatch_aio` (moved in 2026-06-30). The project-intrinsic parts can later be promoted into its own `AGENTS.md` (firstmate-style), keeping fleet/orchestration concerns in frigate.
