# POD2-00960 — briefing for the next instance

Written for you, not for the user. Dense on purpose. Dead ends are marked so you
don't re-walk them.

## The ask

Illustrate an IG post claiming: a Dubai home shut up for a summer breeds mould in
the AC, and switching the system back on puts it all in the air "within the hour".
Constraint: **all data via the production MCP server only** (`https://mcp.gaia-solutions.earth/mcp`),
which doubled as a stress test of that server. Client consented; device **POD2-00960**,
living room.

## Identifiers you'll need

| Thing | Value |
|---|---|
| Device | `POD2-00960` (stored upper-cased in `Device_Measurements.Device_ID`) |
| Old home (mould-plagued) | `4a9b433e-f36b-1410-8ae9-00c3bfb8c933` — 2023-10-26 → 2025-01-16 |
| New home | `f3b48bad-b196-46f5-8163-ed2e58d85920` — 2025-01-16 → current |
| Test property (excluded) | `a17b44af-dc6f-4b8c-baa4-323943dcef6c` — user `90224cba-f126-45c7-a991-f8947ed5f1be` |
| Auth | `x-api-key: $GAIA_MCP_API_KEY`, holds `devices.read.privileged` |

Measure codes: `mold_odor` (ou), `env_temp` (°C), `env_rh` (%), `co2_co2` (ppm),
`sound_leqa` (dB(A)). Tool enums differ from DB codes — it's `mould_odour` and
`sound` in the tool schema. 30-min grain, `Measured_At` is **UTC**; Dubai is UTC+4,
no DST.

## What we built (all merged to main, deployed)

The MCP server could not answer this question at all. Two commits of fixes plus one
of new capability:

1. **Four tools were broken in prod.** `get_jobs`, `get_clients`, `get_reports` all
   died in DI — those repos declare *optional* collaborators and the container
   matched bindings by truthiness, so binding `null` still threw. Fixed to match by
   presence, then bound `undefined` explicitly. `get_clients` had a second bug:
   `sp_findClients` was an unimplemented stub; now resolves via `User.isClient()`.
   `get_process_instances` emitted a bare `WHERE` when unfiltered.
2. **Scoped mode.** All five `iaq_*` tools take optional `device_id` / `property_id`,
   gated per-call on the new `devices.read.privileged` ACL. Scoping lifts the fleet
   guardrails (min-rooms suppression, coverage floor, the 18-hour completeness gate —
   that last one would delete every day a device was offline overnight) and switches
   percentiles from **across rooms** to **across readings**, because one room makes
   p10=p50=p90. `provenance.unit_of_analysis` tells you which you're reading.
   `provenance.room_tenure` gives the allocation history keyed by owning Property_ID.
3. **`day` bucket** on `iaq_seasonal_profile`, keyed yyyymmdd on **Dubai local time**
   (`IAQ_LOCAL_UTC_OFFSET_HOURS`). Other buckets stay UTC so no fleet figure moved —
   verified by diffing `iaq_distribution` against prod: byte-identical bar the new field.
   ⚠️ `iaq_diurnal_profile` still buckets hour-of-day on **raw UTC** — subtract 4 to get
   Dubai local. Pre-existing, not fixed, worth a ticket.
4. **Test accounts excluded** from every `iaq_*` path via `METADATA.TEST_ACCOUNT`
   (`'test.account'`, on the *user*, latest revision wins). Fleet co2 rooms went 33→32.

## Data corrections the user made mid-analysis

Don't re-derive these — they're done:

- The device's first "tenancy" was a **test property**, which read as a house move that
  never happened. Now flagged and filtered.
- The allocation ledger had **one continuous tenancy** and no record of the real
  2025-01-16 move; the address had been edited in place. User created the missing old
  property and split the allocation at `2025-01-16 00:00:00`. Verified: no reading is
  orphaned, old home's last reading 2024-11-29, new home's first 2025-01-30.

## Method notes — what works, what doesn't

- **`sound` is useless as an occupancy signal.** Daily median spans 43.3→50.0 dB over
  three years. Sound *amplitude* (p90−p10) works: ~13 dB occupied, **0.3–0.6 dB** empty.
- **CO₂ is the occupancy detector.** Empty ⇒ decays to outdoor baseline (~400 ppm) and
  the daily swing collapses. Rule used: `co2_p50 < 520 AND co2_swing < 220`. Season- and
  home-independent, unlike temperature (the old home ran ~24 °C, the new one ~27 °C, so a
  fixed temperature threshold misses winter absences — I made that mistake first).
- **Absences sometimes appear as data gaps** (wifi off, e.g. the 57-day hole in summer
  2025) and sometimes as continuous flat data (summer 2026, Dec 2023). Check both.
- Reporting is otherwise good: 645 days of data, median 48/48 readings per day. The
  user's worry about nightly wifi gaps barely shows.

## Findings

**The headline event: 2023-12-06 → 12-19, old home.** Power was continuous across the
whole arc — **48/48 readings on every one of the 25 days from 28 Nov to 22 Dec**, no gap,
no reboot. That is what rules out the burn-in artifact described below, and it is the
first thing to re-check if anyone questions this event.

| Phase | mould p50 | temp | co2 | sound swing |
|---|---|---|---|---|
| Lived in (Nov 20 – Dec 6) | **19** | 24.6 | 530 | ~13 dB |
| Empty, AC off (13 days) | **271** (×14) | 26.3 flat | 403 | 0.3–0.6 dB |
| AC switched back on | **876 peak** (×47) | 20.3 | 408 | — |

Two rates, both hourly-verified:

- **Build-up: 5.6 ou at 04:00 → 267.8 at 08:00 on 6 Dec = ×48 in four hours.** She left
  ~05:00 (noise spike to 50 dB, then the floor). The 13-day plateau was reached within
  ~3 hours of the door closing. The smell does not need a fortnight.
- **Return: 213 ou at 19:00 on 18 Dec → 876 at 06:00 on 19 Dec = ×4.1 in eleven hours.**
  AC on at 21:00 (temp 25.7 → 20.3, noise 41.8 → 55.7). Mould went 213 → 412 in the
  first hour.

**Nine absences detected at the old home**; the mould response is not consistent across
them (5 rise on return, 2 fall, 7 flat). December 2023 is the one with the full arc.

### June 2025 — usable, and worth keeping

The one part of summer 2025 that survives. **Jun 1–11 2025, new home**, sitting on a
complete May (1,483 of 1,488 readings), so there is **no power gap and the settling rule
does not apply**. Median **72–187 ou** against a May baseline of ~9, and the trace is
smooth (hour-to-hour volatility 0.03–0.17, versus 0.01–0.06 for the verified-clean
December plateau at similar levels).

**It is independently corroborated: the client filed a `ux.issue.1` on 2025-06-11**, in the
middle of that stretch. A resident complaining while the index reads 15× its own baseline
is the closest thing we have to ground truth that this index tracks something people can
actually smell — valuable precisely because the scale is uncalibrated.

⚠️ Exclude **2025-06-10** from any summary: it is a `gold.monthly.1` service visit and the
highest June day (187). Could be the technician disturbing the system, could be the
plant-based products registering on a MOX board. Don't try to explain it, just drop it.

### 2024 — the richest seam, 7 of 8 absences usable

Jan–Sep 2024 is essentially unbroken at the old home (Feb, Apr, May, Jun, Aug, Sep are
*complete* months). Eight absences detected; only the Jan 4–10 one is rejected (power gap
in the window). Two standouts, both verified continuous-power, both with plateau volatility
of **0.02–0.08** — as clean as the December data:

| | **29 Mar – 9 Apr 2024** (12 d) | **9 – 19 Jul 2024** (11 d) |
|---|---|---|
| AC during absence | **off** — temp climbs 25.2 → 27.7 | **left ON** — temp flat at 24.3 |
| co2 during | 392–419 (outdoor) | 404–451 (outdoor) |
| noise swing | 0.7–1.2 dB | 1.1–1.8 dB |
| mould before | 6 | 12 |
| mould during | **95** (×16) | **78** (×6.5) |
| on return | 16 | **~200 sustained 20 h**, then 8 |

**July 2024 is the summer analogue of the December event** — same shape, same home, in a
Dubai July. ⚠️ There is a ~4 h data hole right at the return (19 Jul 21:00 → 20 Jul 01:00),
so the 552 ou reading at 01:00 sits inside the 2 h settling window and **is not citable**.
The sustained ~200 from 03:00–21:00 is well outside it and is the usable figure.

Note July 2024 had the **AC running throughout the absence** and the odour still rose ×6.5.
So the build-up is not simply "no airflow" — an empty mould-affected home concentrates it
either way.

### 2026 summer — AC LEFT ON, and it worked

⚠️ **Correction to an earlier pass, which called this "AC off". It is AC ON.** The room
holds **26.9 °C dead flat for 27 days in a Dubai July** (outdoor 41–43 °C by day, 32–34 °C
at night). A sealed unconditioned flat would sit in the mid-30s; holding 27 requires active
cooling. Reading the 25.9 → 27.0 rise as "no cooling" inverts the conclusion — don't repeat it.

**7 Jul – 3 Aug 2026, new home, 27 consecutive days, continuous data, AC running at ~27.**
Departure is visible to the hour (7 Jul, 08:00–10:00: temp 26.4 → 30.1 as the doors open,
co2 1220 → 628, noise 53.5), then the setpoint pulls it back to 27 and holds.

**RH held at 46%. Mould stayed at 2.7–6.4 the entire time.** No build-up at all.

## How this maps onto the summer series

The series makes two claims. Both are supported, and one is close to a controlled proof.

**Claim 1 — "Never off. Dry mode, or 24–25 °C."** The strongest evidence is a *within-home*
comparison (same flat, same established growth, so no building confound):

| Same old home | AC | mould rise |
|---|---|---|
| Dec 2023 | **off** | **×14** |
| Mar–Apr 2024 | **off** | **×11.5** |
| Jul 2024 | **on @ 24.3** | **×6.7** |

Leaving the system running roughly **halved** the build-up in the same building.

**Claim 2 — "If it still smells, the growth is established and drying won't fix it."**
July 2024 is near-controlled: the AC ran at **24.3 °C — exactly the recommended setpoint** —
for 11 days, holding RH at 54%, and the odour *still* rose ×6.7 and hit ~200 ou on the
return day. Doing everything right did not fix it. This is the recommendation stated by the
data almost verbatim, and it is the single most quotable result in the dataset.

**The counterfactual — Jul 2026.** Same client, same behaviour, longer absence, hotter
month, clean system: RH 46%, odour flat at 5. So an empty home does not *create* the smell;
it reveals what is already in the system.

| | Dec 2023 old | Mar–Apr 2024 old | Jul 2024 old | **Jul 2026 new** |
|---|---|---|---|---|
| days empty | 13 | 12 | 11 | **27** |
| AC | off | off | **on @ 24.3** | **on @ 26.9** |
| RH during | 50–61% | 57% | 54% | **46%** |
| mould before → during | 19 → **271** ×14 | 8 → **93** ×11.5 | 11 → **77** ×6.7 | 6 → **5** ×0.9 |
| on return | **876** | 16 | **~200** | not yet |

⚠️ **The "not higher" qualifier is being dropped from the series — DECIDED, 2026-08-07.**
The advice stays a **setpoint** ("dry mode — or 24–25 °C"), because a setpoint is the only
thing a resident can actually set. What goes is the claim that anything above 25 is worse,
because this data does not support it and mildly cuts the other way:

| | setpoint | resulting RH | outcome |
|---|---|---|---|
| Jul 2026, new home | **26.9 °C** | **46%** | no build-up |
| Jul 2024, old home | **24.3 °C** | **54%** | ×6.7 build-up |

Setpoint and building are confounded — the old flat had a far heavier moisture load — so
the honest reading is that **the setpoint is the control, RH is the outcome, and how much
RH a given setpoint buys you depends on the building**. Do not restate the recommendation
as an RH target: residents cannot set RH, and we would be handing them an instruction they
cannot follow. Keep it as the setpoint, and keep RH as what we measure to check it worked.

The runtime physics in post 2 (lower setpoint → more runtime → more water stripped out)
remains correct as mechanism — just never cite *our* measurements as proof of the number.

⚠️ The 227 ou on 2026-08-06 is **5 readings straight after a 2-day gap** — a settling
artifact, must not be used. ⚠️ Reporting has gone intermittent in August 2026 (3 Aug: 29
readings, 6 Aug: 5, 7 Aug: 2). If anyone wants the 2026 return captured live, the device's
connectivity needs checking *before* she gets back.

### Summer 2025 week by week (10 Jun → 2 Oct), settling applied

Only **18 of 115 calendar days have any data** — the outages dominate. Volatility must be
judged against the level (a weak signal is naturally noisier): verified-clean reference is
**0.01 at 120–400 ou**, ~0.35–0.53 at 15–120 ou, ~0.40 below 15 ou.

| Week | Days | Mould median | Volatility | Read |
|---|---|---|---|---|
| 09 Jun | 2 | **187.1** | 0.03 | usable — high and smooth |
| 23 Jun | 1 | 268.7 | — | 2 h only, inside settling |
| *57-day blackout* | 0 | — | — | device dark, absence unmeasurable |
| 18 Aug | 2 | 89.0 | 0.53 | suspect — 3–7× noisier than June at the same level |
| *12-day blackout* | 0 | — | — | |
| 01 Sep | 2 | 6.5 | 0.59 | usable |
| 08 Sep | 4 | 5.2 | 0.37 | usable |
| 15 Sep | 4 | 8.1 | 0.95 | suspect |
| 22 Sep | 1 | 6.8 | 0.41 | 6 h only |
| 29 Sep | 2 | 6.7 | 0.20 | usable |

**The arc is 187 → (blackout) → 89 → (blackout) → ~6, and it stays at ~6 for months**
(monthly p50: Oct 5.8, Nov 7.9, Dec 3.4). Both step-downs happen *inside* blackouts, so
neither is attributable.

⚠️ **The 2025-10-02 cleaning cannot be evaluated** — last reading before it is 30 Sep, next
is 15 Oct. And the level was *already* ~6 for the whole of September, a month before the
visit. Whatever resolved this home's mould problem, the October cleaning was not it. Do not
let anyone build an efficacy claim on that job.

## Things that are NOT true — don't rebuild these

- **"Mould builds up over weeks."** It plateaus in hours and then slowly *decays*.
- **"Gap length predicts the return spike."** It doesn't: 61-day gap → 38 ou; 2-day gap
  → 227 ou. Resumption days run ~2× the following days regardless — a MOX power-on
  artifact (median 31.6 vs 14.7 across 20 resumptions).
- **The summer-2025 episode is unusable — but NOT because the sensor broke.** An earlier
  pass called 2025-09-07 a permanent re-baseline; that is wrong. The same sensor read
  p50 227 / p90 291 on 2026-08-06, so it was never stuck low.

  It is a **power-on stabilisation transient**. The device was dark 12 days (25 Aug –
  5 Sep) and came back at 18:00 on 6 Sep; the odour channel then swung
  356 → 122 → 37 → 192 → 176 → 129 → 114 → 112 → 116 → 666 → 427 → 49 → **2.3** over
  13 hours and was stable afterwards.

  Venting was tested as an alternative and **rejected**: co2 ran 780–1230 all day and
  1291 overnight (a sealed, occupied room — Dec 2023's actual venting sat at 428–489
  while equally noisy), and absolute-humidity range was 1.1 g/m³ vs 3.0 when vented.

  ⚠️ **Operating rule — measured, not assumed.** Settling time was measured across six
  resumption events (hours from first reading until the odour value drops to the
  following day's stable level and stays):

  | gap before | first hour back | settled after |
  |---|---|---|
  | 2 d | 164 ou | **13 h** |
  | 6 d | 310 ou | 1 h |
  | 9 d | 196 ou | 1 h |
  | 12 d | 356 ou | **13 h** |
  | 31 d | 244 ou | 2 h |
  | 33 d | 651 ou | 1 h |

  **Gap length does not predict settling time** (a 33-day gap settled in 1 h; a 2-day gap
  took 13 h). What IS universal is that the *first hour back* is wildly elevated — 164–651
  ou in every single case — then collapses.

  So: **always discard the first 2 h; treat anything inside the first 12 h after a power
  gap as unreliable rather than as signal.** Median settling is ~1–2 h, worst observed 13 h.
  An earlier pass said "12–18 h" as a blanket rule — that over-generalised from the two
  worst cases.

  This explains the resumption-day statistic (median 31.6 vs 14.7 on following days) and
  the 227 on 2026-08-06 (5 readings, straight after a 2-day gap). The Aug-2025 "return
  spike" of 113.9 falls inside this window — the device had returned from a 57-day outage
  that same evening.

  **Literature check** (Ellona publishes no durations; POD2 is documented only as "MOS
  sensors + smart algorithms"). Two distinct layers, and our two numbers match one each:
  hardware warm-up is minutes-to-hours (ENS160 3 min, CCS811 20 min run-in, Figaro TGS2602
  >2 h after a week unpowered) — that's our 1–2 h typical; algorithm baseline re-learning
  is ~12 h–2 days (Sensirion SGP30 up to 12 h to re-establish baseline, SGP40/41 VOC-Index
  default learning time 12 h, CCS811 48 h burn-in, ENS160 largest resistance drift in the
  first 48 h) — that's our 13 h worst case. Commercial MOX stabilisation is often quoted
  at ~24 h generally.

- **Ellona's index is NOT fast-adaptive — worth knowing.** Sensirion's VOC Index is
  explicitly relative to *the past 24 h* (100 = 24-hour average). If Ellona's `mold_odor`
  behaved that way, the 13-day plateau at ~270 would be impossible — a self-recentring
  index would fall back within a day or two. Ours held for 13 days with only a slow decline,
  so the scale is not re-based on recent history. This is what licenses comparing December's
  "empty" level against the same month's "lived-in" level.

- **The discriminator to use, generally:** a real air event moves *several* channels
  together. Sept 2025 has only the odour channel unstable while temp/RH/AH/co2/noise stay
  smooth. Dec 2023 has odour, temp, co2, RH and noise all moving in concert — that is what
  makes it a room event, not a sensor event.
- **Summer 2026 shows no mould build-up** during 27 confirmed empty days (p50 4.4, the
  lowest in the record) — but the AC was off and the air still, so there was nothing to
  carry it to the sensor. Consistent with the story, not evidence for it.

## Deliberately excluded from the illustration

After 14:00 on 2023-12-19 she **opened all windows and doors**, and the reading collapsed
227 → 33 in an hour. Corroborated three ways: co2 stayed at 428–489 while noise was at
full occupied level (this room reaches 556–639 when occupied and closed); absolute humidity
started swinging 3 g/m³/day vs 0.4–0.9 sealed; the room tracked the outdoor diurnal curve
(18.7 °C / 74.6% RH at 06:00 indoors — impossible with an AC running). **That's a winter
response and contradicts the summer advice in the caption**, so the chart is cut at 14:00.
If the post ever goes seasonal, this is a strong second story.

## Caveats that must survive into the post

`mold_odor` is an **uncalibrated ML index** in odour units (MOX board + trained model),
not a concentration. Provenance stamps `absolute_levels_uncalibrated: true`. Speak only
in relative terms — "×47 its normal level", never a number with a unit implying calibration.
The "poor" band floor is 100 (from `defaultAlertProfile`), which is the only numeric
threshold checked into code. n=1 home, n=1 absence.

## Files

| File | What |
|---|---|
| `pod2-00960-hourly-full.csv` | 813 hourly rows, 20 Nov – 23 Dec 2023, all parameters + absolute humidity + phase |
| `pod2-00960-illustration-timeline.csv` | 447 rows, exactly the chart series, 1 Dec → 19 Dec 14:00, with `hours_from_departure` |
| `pod2-00960-illustration-phases.csv` | 3 rows, the phase summary behind the stat tiles |
