# Fleet — in-flight work   ·   crew = live

Phases: Plan Build House Test Review Valid Merge Ship  (●done ◉now ○todo)

 Crew     Ticket                                Summary               Project   Phase            Status
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Victor   KB-51809                              Recalc re-adding an…  snd_aio   ◉○○○○○○○ Plan    🔴 feedback
                                                                                ↳ plan gate
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Echo     KB-49795                              Freight txn safety    snd_aio   ●●●◉○○○○ Test    🔴 feedback
   ↳ In Progress                                                                ↳ draft #1555 green
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Charlie  KB-48976                              Reject rounding (fl…  snd_aio   ●●●●◉○○○ Review  🔴 feedback
   ↳ In Progress                                                                ↳ draft #2150
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Charlie  KB-48976                              Accessorial rejecti…  snd_aio   ●●●●◉○○○ Review  🔴 feedback
   ↳ In Progress                                                                ↳ draft #2150
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Sierra   KB-49466                              Depot rename doesn'…  snd_aio   ●●●●◉○○○ Review  🟡 idle
   ↳ In Progress                                                                ↳ #2113 green
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Zulu     KB-49635                              Auto-sequence TW tu…  snd_aio   ●●●●◉○○○ Review  🟡 idle
   ↳ In Progress                                                                ↳ draft #2142
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Upsilon  KB-52109                              Blocking TW flags s…  snd_aio   ●●●●◉○○○ Review  🟡 idle
   ↳ Ready for Review                                                           ↳ #2143
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Alpha    KB-52256                              TW Order Movements …  snd_aio   ●●●●●●◉○ Merge   🟡 idle
   ↳ Ready for Review                                                           ↳ #2078 · burner ethanol
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Bravo    agent-ticket-flow                     MOM agent ticket fl…  bb_tools  ●●●●●●●◉ Ship    🟡 idle
                                                                                ↳ mom 8c1c9b806
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Omega    loves-test-scout-off + demo-scout-on  Turn off Scout in l…  kbr       ●●●●●●●◉ Ship    🟡 idle
                                                                                ↳ demo Scout live (Super User)
 ─────────────────────────────────────────────────────────────────────────────────────────────────────────────

## Needs feedback — captain

❓ **Echo · KB-49795** — E2E covers the invoice surface where the change lives; cover the other freight
   datasets too (a burner each), or is that enough?

🔴 **Victor · KB-51809** — fix: on recalc, reject open SPRs whose FLI is gone (flag paid ones out-of-sync).
   Approve? And one-off cleanup of existing open dupes, or let next recalc clean them?

❓ **Charlie · KB-48976** — Paused: captain verifying mom; keep draft #2150 as is?

❓ **Charlie · KB-48976** — Paused: captain verifying mom; keep draft #2150 as is?
