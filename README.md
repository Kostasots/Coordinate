# Coordinate

**Officiating operations scheduling — by Sonma Thought Lab**

Coordinate assigns officials to games across many fields and long days, respecting
skill tiers, fatigue limits, and pairing rules. Built first for field hockey
tournaments; the domain rules are isolated so the same engine adapts to soccer,
lacrosse, or parks & rec leagues.

This repository currently contains the **assignment engine** — the part that holds
all the domain logic. The API and web interface sit on top of it and are next.

---

## What it does

1. **Parses** a tournament master schedule (xlsx/csv) into normalized games.
2. **Plans shifts** so no official exceeds a fatigue ceiling.
3. **Assigns** officials to games under hard and soft constraints.
4. **Flags** every compromise it had to make, with a plain-English reason.

---

## Two findings from the real data

Both changed the design, so they're documented rather than buried.

### There is no density valley

The original plan was to split the day at the lowest-density hour. Profiling six
years of SSTG schedules showed that hour doesn't exist:

| 2023 Day 1 | 8a | 9a | 10a | 11a | 12p | 1p | 2p | 3p | 4p | 5p | 6p | 7p | 8p | 9p |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| games | 19 | 18 | 19 | 19 | 19 | 18 | 26 | 19 | 19 | 19 | 19 | 19 | 19 | 17 |

Every field is booked every hour. Valley detection would return rounding noise.
Coordinate splits on **cumulative workload** instead: it computes the minimum
number of shifts that keeps everyone under the ceiling, then places boundaries so
each shift carries a comparable share of the day's slots.

A 14-hour day cannot be covered by two humane shifts. It needs three.

### The 12-hour days were arithmetic, not sloppiness

SSTG 2023 day one is 263 games — **468 official-slots**. At a 1-on-1-off cadence,
one official on a 5–6 hour shift covers 3–4 games. That day needs roughly **179
officials**. A typical roster is ~120.

Nobody scheduled badly. The roster was ~60 people short, and the only variable
left to stretch was human hours.

This shaped the engine's most important behavior: **an official works one shift
per day.** When the roster can't cover the day, Coordinate leaves games flagged
`unfillable` rather than quietly recycling someone into a second shift. An early
version did the recycling and reported a comfortable 6-hour shift while the actual
human stood on a field for 14 hours. Hiding the shortfall is worse than showing it.

Coordinate's most valuable output may not be the assignment sheet. It's the
sentence it can give you in September: *this schedule requires 179 officials, you
have 120, here are the games that will go uncovered.*

---

## Results on real data

SSTG 2023, three days, 659 games:

With 7v7 crewed at two officials, day one is **526 official-slots**, not 468.

| Roster | Unfillable | Longest day | Median day | Max games/day |
|---|---|---|---|---|
| 180 | 107 games | 6.0h | 4.0h | 5 |
| 200 | 22 games | 6.0h | 4.0h | 5 |
| **210** | **0** | **6.0h** | **4.0h** | **5** |

At 210 officials, both tournaments cover fully with **no D+D pairings on
supervised pools, nobody over the 6.5-hour ceiling, nobody past the 5-game cap,
and every new umpire on a small-sided field — zero on full-field games.**

The remaining flags are advisory: forced back-to-backs (mitigated by keeping the
stronger official on the field for continuity) and grade fallbacks.

---

## Sports currently defined

Nine profiles, all on the same 1–4 + N grade scale so a grade means the same
thing everywhere:

| Sport | Crew templates | Notes |
|---|---|---|
| Field Hockey | Full field (2), Small-sided (2, development) | Levels are pool letters A–P |
| Basketball | 3-person (varsity), 2-person (JV), 2-person (rookie dev) | Referee role has a grade floor |
| Soccer | 3-official, 2-official (dual), 2-official (dev) | Center Referee has a grade floor |
| Volleyball | R1+R2+2 line judges, R1+R2, R1 only, R1 only (dev) | Line judges are their own lower-floor role |
| Softball | Fast pitch 3/2, Slow pitch 2/1, Slow pitch 1 (dev) | `backToBack: 'allowed'` by default |
| Baseball | 3 umpires (plate+2 bases), 2 umpires, 1 umpire, 1 umpire (dev) | Plate role has a grade floor; `backToBack: 'allowed'` by default |
| Flag Football | 2-official (Referee+Umpire), 1-official, 1-official (dev) | No 3-official crew modeled — flag rarely runs one |
| Lacrosse | 3-official (Referee+Umpire+Field Judge), 2-official, 2-official (dev) | Referee grade floor is a design choice, not a rulebook requirement |
| Wrestling | 1 referee per mat, always — championship-bracket template just carries a higher grade floor | Structurally different from every other sport here: crew size never scales, only who gets which mat does |

Only field hockey has been run against real tournament data end to end. The
other eight are built the same way — real crew templates, real eligibility
tables, loadable through `rules.loadSport()` — and each has at least one
targeted synthetic test proving its distinguishing mechanic actually works
(basketball's referee grade floor, wrestling's single-official-per-mat with a
championship-only grade floor, baseball's asymmetric plate role), but none has
been checked against a messy real spreadsheet the way field hockey has against
SSTG 2023/2024. Treat them as structurally correct, not field-validated.

## Sports and crew templates

The first screen is a sport picker. But a sport can't answer "how many
officials" on its own — basketball is 2-person or 3-person depending on level,
fast pitch is 2 or 3, field hockey is 2 everywhere. So a sport profile offers a
**menu of crew templates**, and the upload flow assigns one to each division.

Crew templates carry named **roles**, not just a count, because some sports have
asymmetric crews. Basketball's Referee is the lead official and needs a minimum
grade; the Umpires are peers. Field hockey's two umpires are interchangeable. If
crew size were a plain integer, that distinction would have nowhere to live, and
retrofitting it after the database is shaped around a number is painful.

```
Field Hockey
  full_field    2 umpires   Full field (11v11)
  small_sided   2 umpires   Small-sided (7v7)      [development ground]
  unofficiated  0           Showcase / skills

Basketball
  three_person  3 officials Referee (min grade B) + 2 Umpires
  two_person    2 officials Referee (min grade C) + 1 Umpire
  two_person_dev 2 officials Referee + 1 Umpire     [development ground]
```

A template marked `developmentGround` is where the entry tier belongs. Entry-tier
officials are assigned there and **nowhere else** — the assignment is the
training. In field hockey that's the 7v7 fields; in basketball, youth games.

`packages/engine/sports/basketball.js` exists to prove the abstraction holds. A
synthetic 2-day, 6-court basketball tournament runs through the same engine with
correct crew sizes per division, zero rookies on varsity, and every Referee role
meeting its minimum grade.

Adding a sport is adding one file. `rules.js` contains no sport knowledge.

---

## Domain rules

These are the **field hockey** profile's rules. Other sports define their own.

Grade hierarchy: **1 > 2 > 3 > 4 > N** (1 strongest). Originally lettered A–D;
switched to numbers because field hockey pools are *also* lettered (A–P), and
"a Grade A umpire on pool A" reads as one relationship when the two scales are
completely unrelated. Numbers don't collide with anything.

**N ("new")** is the entry tier. New umpires work small-sided (7v7) games only —
the smaller field is the training ground, and the schedule is the development
tool. N is not a weaker 4; it's a different job. The engine will not place an N
on a full-field game under any configuration.

| Pool tier | Pools | Allowed | Fallback | Never |
|---|---|---|---|---|
| Premier | A, B | A | B | C, D |
| Strong | C | A, B | C | D |
| Mid | D, E, F | B, C | A, D | — |
| Developmental | G, H, I | C, D | A, B | — |
| Supervised | J, K, L | A, B, D | C | D+D pairing |
| Overflow | M, N, O, P | C, D | A, B | — |

**Age eligibility:** D-grade officials do not work U19. N works U12/U14
small-sided only. A, B, C work any age band.

**Crew size:** both 11v11 and 7v7 run **two** officials. Small-sided games are a
training ground, not a lighter workload. Showcase and skills sessions need none.

**Max games per person:** a hard cap, set by the assignor at upload (default 5).
Treated as inviolable, like double-booking — if honoring it leaves a game short,
that shortage is the honest answer and gets flagged rather than quietly handing
someone a sixth game.

### Assigning crew templates to divisions

**Field names tell you nothing about format.** An earlier version of this engine
inferred that lettered fields (`15A`, `16C`) were subdivisions of a full pitch and
therefore small-sided. That was wrong — they are real, distinct field locations,
and any division can be scheduled on one. Three U19 games sit on lettered fields
in the 2024 file and are entirely legitimate.

The real signal is the **division** — but the division string isn't always
explicit either. SSTG 2023 writes `U12 (7v7)`; SSTG 2024 writes the same games as
plain `U12`. So Coordinate doesn't guess. `proposeDivisionTemplates(sport, rows)` returns
every division in the schedule with a suggested crew template, a confidence
level, and the reason, and the assignor confirms before anything generates:

| Division | Suggested | Confidence | Games |
|---|---|---|---|
| U16 | full_field | high | 210 |
| U14 (11v11) | full_field | high | 75 |
| U12 | small_sided | **low — confirm** | 60 |
| U19 | full_field | high | 195 |
| U14 (7v7) | small_sided | high | 60 |
| GK/Shooter Showcase | unofficiated | high | 8 |

The confirmed map becomes `config.divisionTemplates`. Anything low-confidence is
surfaced for a human decision rather than assumed. A sport with no naming
conventions to match on (basketball) returns every division as low-confidence,
which is the honest answer — it genuinely can't tell varsity from JV mechanics
without being told.

---

## Running it

```bash
node scripts/demo.js 2023 --roster 210 --max-games 5
node scripts/demo.js 2024 --roster 210 --max-games 5
```

```js
const { runTournament } = require('./packages/engine');

const result = runTournament({
  rows,      // schedule rows, already read from xlsx/csv
  umpires,   // [{ id, name, grade, availability? }]
  config: {
    sport: 'field-hockey',
    maxGamesPerUmpire: 5,
    divisionTemplates: { 'U12 (7v7)': 'small_sided', U16: 'full_field' },
  }
});
```

### Config

| Option | Default | Meaning |
|---|---|---|
| `maxShiftMinutes` | 390 | Fatigue ceiling (6.5h) |
| `targetShiftMinutes` | 300 | What shift planning aims for (5h) |
| `maxIdleMinutes` | 180 | Longest tolerable gap between games |
| `minRestMinutes` | 60 | Required break between assignments |
| `travelBufferMinutes` | 15 | Extra rest when changing venue |
| `maxGamesPerUmpire` | 5 | Hard cap per person per day — **ask at upload** |
| `overlapMinutes` | 60 | Handover band between adjacent shifts |
| `gameDurationMinutes` | 60 | Game length |
| `umpiresPer11v11` | 2 | Crew size, full field |
| `umpiresPer7v7` | 2 | Crew size, small field |
| `sport` | `field-hockey` | Sport profile id |
| `divisionTemplates` | — | `{ "U12 (7v7)": "small_sided", ... }` confirmed at upload |
| `allowDoubleShifts` | false | Permit two shifts in one day (off by design) |

---

## Layout

```
packages/engine/
  index.js          entry point; roster→shift distribution, summary
  src/rules.js      sport-agnostic interpreter — contains NO sport knowledge
  sports/           one file per sport: grades, crew templates, eligibility
  src/parse.js      schedule normalization (messy real-world formats)
  src/shifts.js     workload-based shift planning + feasibility
  src/assign.js     the assignment engine and flagging
scripts/demo.js     runs against real SSTG fixtures
```

Adapting to another sport means adding one file under `sports/`. The engine
doesn't know what field hockey is.

---

## Upload-time questions

Two things Coordinate asks before it generates anything, because guessing either
one produces confident nonsense:

1. **Sport** — sets the grade ladder and the menu of crew templates.
2. **Max games per person per day** (default 5) — a hard cap, not a preference.
3. **Crew template per division** — pre-filled from the schedule, with
   low-confidence rows marked for confirmation.

---

## Open questions

- **Multi-venue travel.** A 15-minute buffer is assumed between venues. Rivercity
  to Mary Stratton may need more.
- **Availability format.** The engine accepts declared windows; the intake method
  (form, import, portal) is undecided.
- **Game duration.** Assumed 60 minutes uniformly. Age bands may differ.

---

## Coordinator setup wizard

`packages/web/setup-wizard.html` — open it directly in a browser, no server or
build step required. Walks a tournament coordinator through:

1. **Sport** — a dropdown across all five profiles. Selecting one shows that
   sport's officiating options as read-only chips (development-tier templates
   marked distinctly) so the coordinator knows what's coming before step 4.
   The actual template choice stays per-division, not tournament-wide — a
   tournament can legitimately run 3-person mechanics for varsity and
   2-person for JV in the same event, and locking the whole tournament to one
   crew size at this step would prevent that.
2. Schedule upload (.xlsx/.csv, parsed client-side with SheetJS)
3. Officials roster upload (separate file — most tournaments run two
   independent spreadsheets, not one combined source)
4. Crew template per division — auto-proposed from the schedule, low-confidence
   rows flagged for confirmation, same logic as `proposeDivisionTemplates()`
5. **Facility layout** (optional) — field names populate automatically once
   the schedule is parsed; the coordinator marks any pairs too far apart for
   a back-to-back, same model as `blockedFieldPairs` above
6. Back-to-back policy — **not allowed / avoid-if-forced / allowed**, with a
   sport-specific recommended default (pulled from that sport's own profile,
   not hardcoded to any one sport) and, when "allowed" is picked, a follow-up
   asking for the max-consecutive-games cap
7. Max games per official per day, with an explicit note that this only bounds
   the *automated* first pass — the assignor can still override by hand after
8. A config summary, plus the exact object `runTournament()` would receive

**This is not wired to the engine yet.** There's no API. Step 8 shows the
assembled config as JSON so you can see what would be sent, but nothing runs.
The sport data in the wizard is a hand-copied, commented mirror of
`packages/engine/sports/*.js` — checked programmatically against the real
engine for all five sports (grades, crew template keys, default back-to-back
policy all compared, zero mismatches) and against SSTG 2023/2024 for the
division-proposal logic specifically (zero mismatches there too). It will
still drift if the engine's sport files change and this file isn't updated
alongside them — replacing this block with a real `GET /sports` call is the
first thing the API should do.

I have not clicked through it in an actual browser myself — no headless
browser was available to test with here. What's verified is the logic
underneath (parity with the engine, checked programmatically) and that the
script parses without a syntax error. The DOM wiring on top — drag-and-drop,
the field-pair builder, the step cards marking themselves done — is untested
by me. Worth exercising it yourself before trusting it with a real tournament.

## Known-conflicts timing — deliberately left out of the wizard

Whether to ask "do any officials have known conflicts?" before generating the
first draft, or hold it for after: **held for after, on purpose.** A few
reasons:

- It's not a single yes/no gate — it's potentially dozens of individual facts
  (one official can't start before 10am, another has a 3pm conflict), and
  collecting all of them before anyone has seen a draft is a lot to ask
  up front for a benefit that's mostly hypothetical until there's a schedule
  to check them against.
- The natural moment a conflict surfaces is when someone looks at a real
  assignment and says "I can't do that." That's a repair against a concrete
  draft, not a precondition for generating one — and it matches how this was
  described early on: an assignor learns of a conflict, inputs it, the system
  re-assigns the affected games.
- The engine already has a hook for this that predates the wizard:
  `umpire.availability` (a list of time windows) is read by
  `declaredFor()`/`available()` in `index.js`. A roster upload that includes
  an availability column can carry this in *before* the first pass, for
  officials who already know their constraints — no separate step needed for
  that case.

What doesn't exist yet, and is real future work: a post-generation screen
where a draft assignment is on screen, flags are visible, and reassigning one
official's game re-runs the affected slice rather than the whole tournament.
That's a genuinely different, bigger piece of work than anything in the wizard
today — worth its own pass rather than bolting a partial version onto step 8.

## Back-to-back policy

Three states, set once per tournament (not yet per-division or per-role,
though your volleyball example argues for that eventually):

| Policy | Behavior |
|---|---|
| `forbidden` | Same tier as double-booking. An official who'd need a back-to-back simply isn't a candidate for the second game. Games can come back short-staffed rather than violate it. |
| `discouraged` | Default. Avoided where possible; when unavoidable, the stronger official carries continuity and the pairing is flagged. |
| `allowed` | No rest requirement beyond not double-booking and beyond facility layout (below, which always applies regardless of this setting). Also changes shift-capacity planning — an official who can legitimately work back-to-back needs less of a shift-length cushion, so `allowed` can lower the total roster size a day requires. |

`config.maxConsecutiveGames` (default 3) only applies when `backToBack` is
`'allowed'`. Even with no rest requirement, nobody runs an unlimited streak —
someone hits the cap and the *next* assignment requires an actual break.
Verified with a single-official, one-field, 8-games-in-a-row test: capped at 3,
nobody exceeds 3; capped at 4, nobody exceeds 4.

Field hockey, basketball, soccer, and volleyball default to `discouraged`.
Softball defaults to `allowed` (illustrative — rec softball commonly runs
officials back-to-back on adjacent diamonds — but not confirmed against real
practice the way field hockey's numbers are).

## Facility layout — blocked field pairs

By default, any two fields at the same venue are assumed close enough for one
official to work back-to-back between them. Most tournament sites are compact
enough that this is simply true, and it's why nothing needed to change here
until it came up.

Some sites aren't compact: a turf complex where field 1 and field 12 are a real
walk, a softball site with four diamonds sharing a concession stand but a fifth
diamond a parking lot away. `config.blockedFieldPairs` marks specific pairs as
unreachable in the time available:

```js
config.blockedFieldPairs = [['Field 1', 'Field 12']];
```

This is a **hard constraint, not a preference** — it applies no matter what
`backToBack` is set to, including `'allowed'`, because it isn't about whether
an official is willing to skip a rest; it's about whether they can physically
be in two places. Verified with a deterministic test: one official, one game on
Field 1 at 8am, one game on Field 12 at 9am, `backToBack: 'allowed'`. Without
the block, that official works both. With `blockedFieldPairs: [['Field 1',
'Field 12']]` declared, the 9am game comes back `unfillable` instead — the
engine refuses the transition rather than silently sending someone on an
impossible sprint across the site.

Adjacency (fields that are fine, like 1 and 2, or 1 and 3) needs no
configuration — that's the default. Only the exceptions get listed.

## Next

- REST API around the engine (upload, assign, override, export) — and point
  the wizard's sport data at it instead of the mirrored block
- Admin interface: review flags, override, re-run
- Mobile-first official lookup — search by surname, see your day
- Branding pass

Built by GSI · Coordinate by Sonma Thought Lab
