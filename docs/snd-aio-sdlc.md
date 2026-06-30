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
   Jira readiness heads-up → herdr worktree on `KB-XXX_descriptor` (off `RC`) → ticket to **In Progress**.

2. **Spec + build**
   - Non-trivial work: `brainstorming` (spec + implementation plan) → `subagent-driven-development` (execute).
   - Small bug/issue: skip brainstorming; go straight to `subagent-driven-development`.
   - Wrap new user-facing strings with `tr()` → `backend-i18n`.
   - Fixing bug subtasks on an existing story branch → `fixbugs`.

3. **Housekeeping** — code review, simplification, best-practice check. **[§4.1 TBD — define the stack/order]**

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
   - Open a **draft** PR only: `gh pr create --draft`. **Never a full/ready PR without the captain's approval** (full PRs create review noise for the team). Follow the PR template. **[TBD: template location/sections]**
   - Set the story to **Ready for Review** → `snd-jira-housekeeping`.

7. **Ready for Testing** → `snd-jira-housekeeping`
   Once dev + E2E are done and it's ready for third-party manual validation, move the story to **Ready for Testing**.

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
| 2a Spec / plan | `brainstorming` | have |
| 2b Execute | `subagent-driven-development` | have |
| 3 Housekeeping | `/code-review`, `/simplify`, `requesting-code-review` | have-ish, **§4.1 TBD** |
| 4 Squash + push + CI | **context** (git + `gh run watch`) | ✅ in this doc |
| 5 Unit / E2E testing | `uv run pytest`, `snd_tests`, `gravi-burners` | partial, **TBD** |
| 6 Draft PR | **context** (`gh pr create --draft` + template) | ✅ in this doc; template **TBD** |
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

1. **§4.1 housekeeping stack** — exact review / simplify / best-practice tools + order.
2. **Testing** specifics — the hourglass details; how `snd_tests` E2E suites run against a burner.
3. **PR template** — location + required sections.
4. **Jira** — current-sprint detection + energy-point estimation (manual vs assisted). (Transition/field names are resolved at runtime via `getTransitionsForJiraIssue`.)

*Resolved:* feature-branch base = `RC`; worktree mechanism = herdr `worktree create`.

## Status

- ✅ **Built:** `snd-kickoff`, `snd-jira-housekeeping`; this operating manual (sequence + squash+push + draft-PR as context).
- ▶ **Next:** define **§4.1 housekeeping**, then **testing** specifics + PR template.
- ⏭ **Later:** the **orchestration layer** — the first mate dispatching a crewmate per ticket and supervising it through this playbook (herdr-driven).

## Where this lives

A frigate reference doc for now (the orchestrator points crewmates at it; `snd_aio` is still a placeholder symlink). Once `snd_aio` is properly moved in, the project-intrinsic parts can be promoted into its own `AGENTS.md` (firstmate-style), keeping fleet/orchestration concerns in frigate.
