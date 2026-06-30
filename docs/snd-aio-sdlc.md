# supply_and_dispatch_aio — SDLC

> AIO is the core product. Other projects follow a **similar but not identical** process.
> This is the backbone the frigate orchestrator drives: each KB ticket flows through these
> stages, with the right skill at each. Captured 2026-06-30. **[TBD]** = to define as we build.

## Branch model

- `RC` — Release Candidate
- `beta` — prod for canary releases
- `sharedprod` — prod for all remaining clients

Promotion flows `RC → beta → sharedprod`.

## Per-ticket development procedure

1. **Jira ticket** — AIO project is `KB-XXX`. (Jira rules below. Site `gravitatedxp.atlassian.net`, via Atlassian MCP.)
2. **Worktree + branch**, one per ticket. Convention: `KB-XXX_descriptor` (ticket # + 1–2 word description). **[TBD: herdr `worktree create` vs `.worktrees/` sibling; base branch]**
3. **Development**
   - Spec + implementation plan via `superpowers:brainstorming` (skipped for small bugs/issues).
   - Execution **always** via `superpowers:subagent-driven-development`.
4. **Housekeeping** — code review, code simplification, best-practice check. **[4.1 TBD: define the exact stack/order]**
5. **Squash + push** the feature branch. Squash so the initial dev reviews in one pass. Push triggers CI → per-SHA image (required to boot a burner).
6. **Testing & verification** — "hourglass": heavy unit tests + E2E tests. **[details TBD]**
   - 6.1 E2E + manual verification run against **burners** (ephemeral envs booted off the branch image; load a dataset; run E2E suites). → `gravi-burners` skill.
7. **Draft PR** for final review. **Never open a full PR without approval** (noise for the rest of the team). Observe the PR template. **[TBD: template location/contents]**

## Jira rules

- Every story: **≥2 subtasks — Dev & Validate**.
- Must be in the **current sprint** before it can be PR'd.
- Must have an **energy-point** estimate.
- Statuses: **In Progress** (work started) → **Ready for Review** (PR open) → **Ready for Testing** (dev complete, e2e done; ready for third-party manual validation).
- **AC drift:** acceptance criteria often shift during dev — document adjustments in a **Dev Implementation** section on the ticket (reflect what was *actually* delivered).
- **Test coverage:** document in a ticket comment — the gherkin from `snd_tests` (if applicable) + the unit tests added.

## Stage → skill / tool map

| Stage | Skill / tool | Status |
|---|---|---|
| 1 Jira intake | Atlassian MCP + kickoff skill | **new** |
| 2 Worktree + branch | herdr / git + kickoff skill | **new** |
| 3a Spec / plan | `superpowers:brainstorming` | have |
| 3b Execute | `superpowers:subagent-driven-development` | have |
| 4 Housekeeping | `/code-review`, `/simplify`, `superpowers:requesting-code-review` | have-ish, **4.1 TBD** |
| 5 Squash + push + CI | git + `gh run watch` + new helper | **new** |
| 6 Unit / E2E testing | `snd_tests`, `uv run pytest` | **TBD** |
| 6.1 Burner E2E / manual | `gravi-burners` | have |
| 7 Draft PR | `gh pr create --draft` + template + new skill | **new** |
| Jira hygiene (statuses, AC/dev-impl, coverage, subtasks, sprint, points) | Atlassian MCP + new skill | **new** |
| (bug fixes on a story) | `fixbugs` command | have |
| (i18n strings) | `backend-i18n` | have |

## Naming & skill-vs-context

New AIO skills use the **`snd-`** prefix (product-based, so future projects namespace cleanly):
`snd-kickoff`, `snd-jira-housekeeping`, `snd-testing`, … Cross-project skills stay unprefixed
(`brainstorming`, `subagent-driven-development`, `gravi-burners`).

Only specialized, reusable, triggerable knowledge becomes a **skill**. Mechanical/deterministic
steps — squash+push, "draft PR not full", the branch model, the pipeline sequence — are **context**
in the operating manual, not skills.

Existing AIO-specific skills to retrofit to `snd-` later (noted, not yet done): `fixbugs` →
`snd-fixbugs`, `backend-i18n` → `snd-backend-i18n`.

## Open definitions (nail down as we build)

1. Feature-branch **base** (RC? sharedprod?) and **worktree mechanism** (herdr vs `.worktrees/`).
2. **4.1 housekeeping stack** — exact review / simplify / best-practice tools + order.
3. **Testing** specifics — the hourglass details; how `snd_tests` E2E suites run against a burner.
4. **PR template** — location + required sections.
5. **Jira** — current-sprint detection, energy-point estimation (manual vs assisted), exact transition names.

## Build order (proposed, bottom-up)

1. **Ticket kickoff** — stages 1–2 + set In Progress + hygiene checks (Dev/Validate subtasks, in sprint, points).
2. **Squash + push** — stage 5 + CI image watch.
3. **Draft PR** — stage 7 + template + Ready for Review.
4. **Jira hygiene** — AC/dev-impl doc + test-coverage comment + status transitions.
5. **Testing** + **4.1 housekeeping** — once their stacks are defined.
6. **Orchestration spine** — the operating manual that sequences the above and drives a crewmate per ticket.
