# Fleet — in-flight work   ·   crew = live

Phases: Plan Build House Test Review Valid Merge Ship  (●done ◉now ○todo)

 Crew     Ticket          Summary                                  Project   Phase            Status
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Charlie  KB-49995        Carrier alloc columns                    snd_aio   ◉○○○○○○○ Plan    🔴 blocked
   ↳ To Do                                                                   ↳ plan gate
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Hotel    KB-49849        Turn ETA / route details                 snd_aio   ●●●◉○○○○ Test    🔴 feedback
   ↳ In Progress                                                             ↳ CI watch
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Alpha    KB-48925        Compartment products (TW)                snd_aio   ●●●●◉○○○ Review  🔴 feedback
   ↳ In Progress                                                             ↳ draft #1581
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Delta    KB-48310        Supply Selection (TW)                    snd_aio   ●●●●◉○○○ Review  🔴 feedback
   ↳ In Progress                                                             ↳ draft #1531
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Bravo    KB-39382        Freight line item perf                   snd_aio   ●◉○○○○○○ Build   🔄 working
   ↳ Ready for Development                                                   ↳ plan auto-ok
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Echo     KB-49795        Freight txn safety                       snd_aio   ●●●●◉○○○ Review  📜 captain review
   ↳ In Progress                                                             ↳ draft #1555
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────

## 🔴 Needs feedback — captain
- **KB-48925** (KB-48925_compartment-products) — Heal-on-append vs explicit rebuild for turns that ALREADY exist with zero compartments — the only piece left. Tatum is unblocked without it.
- **KB-48310** (KB-48310_supply-selection) — Scope: which of the 7 confirmed gaps ride here (Delta's pick-two = findings 1 and 2). Blend dataset still needs your mom-auth --apply + a tank-wagons burner.
- **KB-49849** (KB-49849_turn-eta) — R3/R8: cheap swap REFUTED by execution. Only path left is the state-hoist + R14 in the shared route builder — this branch or its own ticket?

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
