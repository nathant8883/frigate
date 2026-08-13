# Fleet — in-flight work   ·   crew = static (herdr skipped/down)

Phases: Plan Build House Test Review Valid Merge Ship  (●done ◉now ○todo)

 Crew     Ticket          Summary                                  Project   Phase            Status
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Delta    KB-48310        Supply Selection (TW)                    snd_aio   ●◉○○○○○○ Build   🔴 feedback
   ↳ In Progress                                                             ↳ volume rule
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Alpha    KB-48925        Compartment products (TW)                snd_aio   ●●●●◉○○○ Review  🔴 feedback
   ↳ In Progress                                                             ↳ draft #1581
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Hotel    KB-49849        Turn ETA / route details                 snd_aio   ●●●●◉○○○ Review  🔴 feedback
   ↳ In Progress                                                             ↳ draft #1562
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Bravo    KB-39382        Freight line item perf                   snd_aio   ●◉○○○○○○ Build   ✗ herdr
   ↳ In Progress                                                             ↳ cache + E2E tests
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Charlie  KB-49995        Carrier alloc columns                    snd_aio   ●◉○○○○○○ Build   ✗ herdr
   ↳ In Progress                                                             ↳ plan ok (captain)
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Echo     KB-49795        Freight txn safety                       snd_aio   ●●●●◉○○○ Review  📜 captain review
   ↳ In Progress                                                             ↳ draft #1555
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────

## Needs feedback — captain

❓ **Alpha · KB-48925** — heal-on-append or an explicit rebuild for turns that already exist with zero compartments?

❓ **Delta · KB-48310** — which of the 7 confirmed gaps ride here? Delta's pick-two is findings 1 and 2.

❓ **Hotel · KB-49849** — cheap R3/R8 swap refuted by execution; state-hoist + R14 on this branch or its own ticket?

## Recently done
 Crew     Ticket          Summary                                  Project   Phase
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Bravo    mom-sprint-hub  Sprint Hub: refresh + rollover pts       bb_tools  ●●●●●●●◉ Ship
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Golf     KB-42954        Seed: backhaul order                     snd_aio   ●●●●●●●◉ Ship
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Bravo    KB-49277        RabbitMQ broker slice 1                  snd_aio   ●●●●●●●◉ Ship
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Foxtrot  KB-45866        Review: Samsara odometer                 snd_aio   ●●●●●●●◉ Ship
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
