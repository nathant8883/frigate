# Fleet — in-flight work   ·   crew = live

Phases: Plan Build House Test Review Valid Merge Ship  (●done ◉now ○todo)

 Crew     Ticket            Summary                                Project   Phase            Status
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Foxtrot  KB-48819          Auto load volumes on assign (TW)       snd_aio   ●●●◉○○○○ Test    🔴 feedback
   ↳ In Progress                                                             ↳ E2E blocked
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Alpha    KB-48925          Compartment products (TW)              snd_aio   ●●●●◉○○○ Review  🔴 feedback
   ↳ In Progress                                                             ↳ draft #1581 · 59/59
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Delta    KB-48310          Supply Selection (TW)                  snd_aio   ●●●◉○○○○ Test    🔄 working
   ↳ Ready For Testing                                                       ↳ #1531 · committing 2 fixes
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Hotel    KB-49849          Turn ETA / route details               snd_aio   ●●●◉○○○○ Test    📜 captain review
   ↳ In Progress                                                             ↳ burner provisioning
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Echo     KB-49795          Freight txn safety                     snd_aio   ●●●●◉○○○ Review  📜 captain review
   ↳ In Progress                                                             ↳ draft #1555
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Bravo    KB-39382          Freight line item perf                 snd_aio   ●●●●◉○○○ Review  🟡 idle
   ↳ Ready for Review                                                        ↳ #1596 · out of draft
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Charlie  KB-49995          Carrier alloc columns                  snd_aio   ●●●●◉○○○ Review  📜 captain review
   ↳ In Progress                                                             ↳ draft #1595
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Juliet   KB-50420          Sites quick search (SSRM)              snd_aio   ●●●●◉○○○ Review  📜 captain review
   ↳ In Progress                                                             ↳ draft #1631 · captain gate
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Kilo     burner-live-fix   burner-live false negative             frigate   ●●●●●●●◉ Ship    🔄 working
                                                                             ↳ ac30a0c committed
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────

## Needs feedback — captain

❓ **Alpha · KB-48925** — equipment borrow fills per piece (a tractor-only shift borrows just the trailer) — keep that, or only borrow when the shift has no equipment at all?

❓ **Foxtrot · KB-48819** — seeder scope: spend ~30-60 lines fixing shared E2E seeding via /driver/create_event, or ship #KB-48819 with the E2E dropped (its own ticket) — your call

## Recently done
 Crew     Ticket            Summary                                Project   Phase
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Golf     KB-50434          Site notes No-Blank filter             snd_aio   ●●●●●●●◉ Ship
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Golf     sprint-o-slides   Sprint O demo slides                   bb_tools  ●●●●●●●◉ Ship
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 India    tw-dataset-blend  TW dataset: blend option               bb_tools  ●●●●●●●◉ Ship
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 India    burner-live       Burner-live detection                  frigate   ●●●●●●●◉ Ship
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Bravo    mom-sprint-hub    Sprint Hub: refresh + rollover pts     bb_tools  ●●●●●●●◉ Ship
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Golf     KB-42954          Seed: backhaul order                   snd_aio   ●●●●●●●◉ Ship
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Bravo    KB-49277          RabbitMQ broker slice 1                snd_aio   ●●●●●●●◉ Ship
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Foxtrot  KB-45866          Review: Samsara odometer               snd_aio   ●●●●●●●◉ Ship
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
