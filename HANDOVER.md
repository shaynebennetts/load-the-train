# Load the train — handover

Project 1, Basic Skills for Experimentalists (TIGP 2026). Brief: `loadthetrainspec.md`
(Shayne's, authoritative).

**Read this file, then only the topic doc you need.** They are separate so you do not have to
load all of it:

| doc | when to read it |
|---|---|
| [`docs/PHYSICS.md`](docs/PHYSICS.md) | before touching the integrator, the parameters or the ledger |
| [`docs/MEASUREMENTS.md`](docs/MEASUREMENTS.md) | before quoting any number, anywhere |
| [`docs/TESTING.md`](docs/TESTING.md) | running the tests, the harness, headless capture |
| [`docs/PUBLISHING.md`](docs/PUBLISHING.md) | the live URLs, Pages, the website, the wall card |
| `SPEC.md` | the full specification and decision record |
| `PLAN.md` | the original build plan; partly historical, task 5 superseded |

---

## 1. Where things stand — 2026-09-12

**Built, playable, published.** One HTML file: no server, no build step, no dependencies, no
network requests. Live at https://shaynebennetts.github.io/load-the-train/ — see
`docs/PUBLISHING.md`.

- All four of the brief's acceptance tests pass, plus a supplementary adhesion check (S3) and
  a determinism check. `app/index.html?test`.
- Physics core, rules, rendering, input, HUD, fill strip, yard map, intro, end-of-run report,
  practice mode and personal-best storage are all in.
- Project website, icon and screenshots at the repository root; card on the class wall.
- **Outstanding:** grade bands (open point 1, needs a decision), 60 fps profiling on a real
  phone (PLAN task 17), cross-browser check (task 25).

> **A correction to the record.** The commit `HANDOVER: published state, measured run lengths,
> grade-band data` claims changes that never reached the file — its edit script aborted on a
> not-found string before writing, and only one open point was actually updated. If you are
> reading that commit message, trust this file instead. The content it described is now here
> and in `docs/`.

---

## 2. Parameter history

Each change is a separate commit with its measurements in the message. **The commit log is the
real record of why every parameter is what it is.**

| change | from | to | why |
|---|---|---|---|
| traction law | `min(throttle·P/v, μmg)` | `throttle · min(P/v, μmg)` | **bug** — `PHYSICS.md` §3 |
| rated power | 3.0 MW | **1 MW** | 3 MW → 30 kW → 500 kW → 1 MW, last step on Shayne's instruction |
| chute flow | 2500 t/h | 100 000 t/h (40×) | accretion term was otherwise unobservable |
| time compression `C` | 25 | **1 — none** | timing windows scaled as 1/C and became unplayable |
| approach | 1000 m | 30 m | 1 km was a minute of holding full power |
| friction brake | `k_b·M·g` | **removed** | retardation is regenerative |
| controls | throttle + brake + reverser | **one signed throttle** | follows from the above |
| pass threshold | 95 % per car | **90 %** | Shayne's call; **departs from the brief**, disclosed in the intro |
| `stopDistance()` | constant-power only | **piecewise** | it under-estimated; `PHYSICS.md` §5 |

---

## 3. Open points

1. **Grade bands are too loose, and this is now a decision, not a measurement.**
   `rules.BANDS` is `A < 150 s, B < 180 s, C < 225 s`, invented and never calibrated. The
   physics floor is **~91 s**; Shayne's first complete run was **131 s** and scored an **A**
   while he described it as "OK". Proposed **A < 105, B < 125, C < 150** — put to him
   2026-09-12, **not yet answered, not applied**. Whatever is chosen, change `rules.BANDS`
   and `SPEC.md` §5.6 together. Numbers in `docs/MEASUREMENTS.md` §1.
2. **Both wall blurbs are Claude's drafts.** The course rule is that the blurb is the
   student's own two sentences. Offered 2026-09-12, not yet replaced. Same for the project
   website prose.
3. **Departures from the brief's own text**, all disclosed to the player in the intro and
   recorded in `SPEC.md` §3.2: 1 MW vs its 2–4.5 MW band, 30 m vs its 1 km, 100 000 t/h vs
   its 1500–3000 t/h, and 90 % vs its 95 % pass threshold. The brief's §3.8 explicitly permits
   raising the flow rate; the others are out-of-band and are stated as such.
4. **Supplementary check S3** is beyond the brief's four tests. Kept because Test 3 runs at
   `μ → ∞` and so never exercises the adhesion cap. Shayne may strike it.
5. **`PLAN.md` task 5 is superseded** (friction brake removed) and annotated as such. Tasks
   22, 17 and 25 outstanding.

**Closed:** the intro physics wording (accepted as written 2026-09-11; the draft bar and
`.draft` CSS are gone) · run length (measured, 90–135 s) · the repository-root prohibition
(the root now holds the project website it was reserved for — `docs/PUBLISHING.md` §2).

---

## 4. Next steps, in order

1. **Get a decision on the grade bands** and apply it. Everything else is polish.
2. Ask Shayne for his own blurb and website wording.
3. Profile on a real phone (PLAN task 17). Physics is 4 steps/frame, so it should be
   comfortable, but it is unverified on hardware.
4. Cross-browser check (task 25), including confirming zero network requests in devtools.

---

## 5. Working agreement with Shayne

- He wrote the brief and is the physicist. When he says the physics is wrong, **check it
  properly and show the numbers** rather than either capitulating or getting defensive. One
  such challenge turned out to be correct behaviour that was merely invisible in play; the
  useful response was measured evidence plus a design fix, not a code change.
- He makes playability calls that override the brief's own parameter bands. Do them, then
  disclose them in `SPEC.md` §3.2 and in the player-facing intro.
- He prefers the simple mechanism over the clever one. A `rank` field added to the class wall
  to control ordering was rejected in favour of just setting the dates — see
  `docs/PUBLISHING.md` §4.
- Commit per change with the measurements in the message.
- **Measure before asserting.** Several documented numbers in this project turned out to be
  stale by 10× because they were carried forward from an earlier parameter set instead of
  being re-measured. `docs/MEASUREMENTS.md` exists so that stops happening.
