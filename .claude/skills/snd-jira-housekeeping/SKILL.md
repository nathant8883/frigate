---
name: snd-jira-housekeeping
description: >-
  Keep a supply_and_dispatch_aio (AIO) KB ticket's Jira state correct through the SDLC — run the
  pre-PR readiness gate (≥2 Dev & Validate subtasks, in the current sprint, has an energy-point
  estimate), drive status transitions (In Progress → Ready for Review → Ready for Testing), document
  AC drift in a Dev Implementation section, and post the test-coverage comment (gherkin from
  snd_tests + unit tests added). Use whenever updating a KB ticket's Jira status, checking whether a
  story is ready to PR, recording what was actually delivered vs the original acceptance criteria, or
  documenting test coverage on a ticket — "is KB-1234 ready to PR", "move this to ready for review",
  "mark it ready for testing", "update the ticket with what we built", "log the test coverage". AIO-specific.
---

# snd-jira-housekeeping — AIO Jira hygiene

Keeps an AIO KB ticket's Jira honest as it moves through the SDLC. Jira: `gravitatedxp.atlassian.net`
(Atlassian MCP tools). Full SDLC: `frigate/docs/snd-aio-sdlc.md`.

Resolve transition and field names **at runtime** — don't hardcode ids. Use
`getTransitionsForJiraIssue` for transitions, and read the issue's fields for sprint and the
energy-point estimate (field names vary by board).

## Tooling

Drive Jira through the **Atlassian (Jira) MCP** — it's available in this workspace; use it rather
than the Jira web UI or asking the captain to do Jira steps by hand. The tools are deferred, so load
them with ToolSearch as needed. Key ones: `getJiraIssue`, `searchJiraIssuesUsingJql`,
`getTransitionsForJiraIssue`, `transitionJiraIssue`, `addCommentToJiraIssue` (coverage comment),
`editJiraIssue` (Dev Implementation section / sprint / estimate), `createJiraIssue` (subtasks). Site
`gravitatedxp.atlassian.net`, project `KB`; get the cloudId via `getAccessibleAtlassianResources` or
pass the site URL as `cloudId`.

## Pre-PR readiness gate (run before opening a draft PR)

A story may **not** be PR'd until all three hold. Check them and **flag any gap to the captain before
PRing** — offer to fix, but never change Jira silently (sprint membership and point estimates are the
captain's call):

- **≥2 subtasks — Dev & Validate.** If missing, offer to create them.
- **In the current sprint.** If not, offer to add it.
- **Energy-point estimate.** If absent, ask the captain for the number.

If a gap is unresolved, **do not open the PR** — report what's missing and stop.

## Status transitions

Drive the story through AIO's statuses (find the transition by name via `getTransitionsForJiraIssue`,
then `transitionJiraIssue`):

- **In Progress** — work started (set at kickoff; see `snd-kickoff`).
- **Ready for Review** — a (draft) PR is open.
- **Ready for Testing** — dev complete and E2E done; ready for third-party manual validation.

## Document AC drift (Dev Implementation section)

Acceptance criteria usually shift during dev. Before review, record what was **actually delivered** in
a **Dev Implementation** section on the ticket — append a `## Dev Implementation` section to the
description; **don't rewrite the original AC**. Call out any deviations from the original AC and why,
so a reviewer/validator sees what changed.

## Test-coverage comment

Add a comment to the ticket documenting coverage so the validator knows what's covered:

- The **gherkin** scenarios from `snd_tests` that cover the feature (if applicable).
- The **unit tests** added — names and what each asserts.

Keep it concise — a coverage summary, not a test dump.
