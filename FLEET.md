# Fleet — in-flight work   ·   crew = live

Phases: Plan Build House Test Review Valid Merge Ship  (●done ◉now ○todo)

 Crew     Ticket            Summary                                Project  Phase            Status
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Foxtrot  KB-48819          Auto load volumes on assign (TW)       snd_aio  ●●●◉○○○○ Test    🔴 feedback
   ↳ In Progress                                                            ↳ shift leak blocks green
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Alpha    KB-48925          Compartment products (TW)              snd_aio  ●●●●◉○○○ Review  🔴 feedback
   ↳ Ready for Review                                                       ↳ #1581 · RC merge
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Echo     KB-49795          Freight txn safety                     snd_aio  ●●●●◉○○○ Review  📜 captain review
   ↳ In Progress                                                            ↳ draft #1555
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Hotel    KB-49849          Turn ETA / route details               snd_aio  ●●●●◉○○○ Review  📜 captain review
   ↳ In Progress                                                            ↳ vent · staged
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Charlie  KB-49995          Carrier alloc columns                  snd_aio  ●●●●◉○○○ Review  📜 captain review
   ↳ In Progress                                                            ↳ rig · ready
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Golf     fleet-board-wrap  Board text wrapping                    frigate  ●●●●●●●◉ Ship    🟡 idle
                                                                            ↳ committed
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────

## Needs feedback — captain

❓ **Alpha · KB-48925** — pre-trip writes equipment numbers without ids — leave Alpha's read-time fallback as
   the fix, or also capture ids at the source?

❓ **Foxtrot · KB-48819** — E2E repeatability needs shared-suite surgery (seed_shift leaks all 8 shifts per
   run) — spend a third cycle, or stop with both approved fixes and write it up?
