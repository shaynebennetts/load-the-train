# Load the train — handover

**Read this first, then `SPEC.md`. `PLAN.md` is the original build plan and is now partly
historical — see §6.**

Project 1, Basic Skills for Experimentalists (TIGP 2026). Brief: `loadthetrainspec.md`
(Shayne's, authoritative). Repo: `github.com/shaynebennetts/load-the-train`,
**nothing has ever been pushed** — all work is local commits on `main`.

State as of commit `c936348`.

---

## 1. Where things stand

**The game is built and playable.** Open `app/index.html` directly from disk; no server, no
build step, no dependencies, no network requests. 115 kB, one file.

- All four of the brief's acceptance tests pass, plus a supplementary adhesion check (S3)
  and a determinism check. Run them at `app/index.html?test`, or click DEBUG on the title
  screen, or `LTT.runTest(n, opts)` in the console.
- Physics core, rules, rendering, input, HUD, fill strip, yard map, intro, end-of-run
  report, practice mode and personal-best storage are all in.
- **Not done:** grade-band calibration (PLAN task 22), 60 fps profiling on a real phone
  (task 17), cross-browser/Pages deployment check (task 25).

---

## 2. What changed after the game first worked

The parameters in `SPEC.md` §3 were substantially retuned on Shayne's play feedback, in this
order. Each is committed separately with its measurements in the commit message.

| change | from | to | why |
|---|---|---|---|
| traction law | `min(throttle·P/v, μmg)` | `throttle · min(P/v, μmg)` | **bug** — see below |
| rated power | 3.0 MW | 500 kW | twitchy, then overcorrected to 30 kW, then settled |
| chute flow | 2500 t/h | 100 000 t/h (40×) | accretion term was unobservable |
| time compression `C` | 25 | **1 — none** | timing windows scaled as 1/C and became unplayable |
| approach | 1000 m | 30 m | 1 km was a minute of holding full power |
| friction brake | `k_b·M·g` | **removed** | retardation is regenerative |
| controls | throttle + brake + reverser | **one signed throttle** | follows from the above |

**The traction bug is worth understanding**, because it was the cause of "control fidelity is
way too coarse". With the throttle scaling only the power term, the `min()` always selected
the adhesion cap below `throttle · v_c`, so **1 % throttle and 100 % throttle both produced
the full 441.30 kN** and the throttle had no authority at all from rest. Now the notch
commands a fraction of available tractive effort, which is what a real locomotive controller
does.

---

## 3. The physics, in one page

Read `SPEC.md` §2 properly, but the load-bearing facts:

- **The integrator state is momentum**, `p = Mv`, not velocity. With grain arriving at
  `u = 0` the variable-mass law is just `dp/dt = F_ext`, so the `v·dM/dt` retarding term
  **is never written down anywhere** — it emerges when `v` is recovered as `p/M`. Do not
  "add" it. If you ever find yourself typing `v*dMdt` into the integrator, something has
  gone wrong.
- **The mass update happens after the mechanical update, deliberately.** That operator split
  is the perfectly-inelastic-collision picture: the mass rises with `p` untouched, so `v`
  drops and exactly `½v²dm` of kinetic energy disappears. That is where the brief's factor
  of two lives.
- **`ledger` must not be written from `physics`'s algebra.** It is handed raw per-substep
  samples and derives everything from `SPEC.md` §2.7 route 2 on its own. This separation is
  the brief's explicit safeguard against the factor-of-two error and is a review criterion,
  not a style preference.
- **There is no loss coefficient.** Loss is the aperture-trough overlap integral and nothing
  else. Integrated over one pitch it is exactly `L_body/L_pitch` = 91.176471 %.
- **There is no brake.** `braking()` does not exist. Retardation is `traction()` with a
  negative argument.

### Current parameters

```
g 9.80665     m_loco 150 t    m_driven 150 t   P_rated 500 kW   mu 0.30
N_cars 20     m_tare 28 t     m_cap 100 t      L_body 15.5 m    L_pitch 17.0 m
mdot 27 777.8 kg/s (100 000 t/h)               w_chute 0.8 m    L_loco 21 m
x_start -30   x_min -60       x_max 700        C_time 1         h_step 1/240 s
fill_target 0.95              loss_budget 0.01                  test_tol 0.001
```

Derived: `M_tare` 710 t, `M_full` 2710 t, `F_adhesion` 441.30 kN, `v_cross` 1.133 m/s,
`train_len` 361 m, `t_fill_car` 3.60 s, `t_fill_all` 72.0 s, `gap_duty` 8.82 %.

### The two numbers that shape the game

```
loading terminal speed   v_eq = sqrt(P_rated / mdot) = 4.243 m/s
one-pass fill limit      L_body / (m_cap/mdot)       = 4.306 m/s
```

At full throttle with the chute open, the grain holds the train just *below* the speed at
which a car can still be filled in one pass. That is a coincidence of the chosen parameters,
not something arranged in code — **if you retune `P_rated` or `mdot`, check it still holds.**
Past about 500 kW the locomotive outruns the chute and the accretion drag stops governing.

---

## 4. Tests — how to run them, and the rules

```
app/index.html?test          panel, all tests
LTT.runTest(3, {mu: Infinity, P_rated: 4e5, M: 5e5, t: 30})     any override
LTT.ledger()  LTT.params()  LTT.state()  LTT.setState({...})  LTT.step(dt)
```

Headless, against the shipped file (used throughout the build). The scratch scripts are not
in the repo; here is the whole harness, which runs the **actual shipped code**, not a copy:

```js
// harness.js
const fs = require('fs');
const APP = 'C:/Claude/TIGP-2026-2/load-the-train/app/index.html';
function loadCore() {
  const m = fs.readFileSync(APP, 'utf8')
              .match(/<script id="ltt-core">([\s\S]*?)<\/script>/);
  if (!m) throw new Error('no #ltt-core block');
  (new Function(m[1]))();          // the core publishes globalThis.LTT
  return globalThis.LTT;
}
module.exports = { loadCore };
// runall.js:  const {loadCore}=require('./harness.js');
//             console.log(loadCore().runAll().text);
```

Screenshots were taken by copying `app/index.html` to a temp file, injecting
`localStorage.setItem('ltt.seen','true')` before `#ltt-core`, pinning `#app` to the target
size (headless lays out ~80 px wider than it screenshots, which silently clips the
right-hand controls), then driving the real UI with `--headless=new --screenshot`.

**Rules, from the brief, which must not be relaxed:**
- Do not weaken a test to make it pass.
- Do not hard-code an expected value from an observed output.
- Do not catch an exception to make a failure disappear.
- If a test fails, report the failure and stop.

**One trap already found and fixed, do not reintroduce it.** `row()` used to fall back to the
raw absolute difference when the expected value was zero, so `ΔKE + E_diss = 0` compared
**joules against a dimensionless 0.1 % tolerance**. It passed only by luck of magnitude, and
broke the moment the flow rate rose 40×. Zero-expectation rows now **require** a `scale` and
`row()` throws without one. Keep that.

---

## 5. Known open points

1. **Grade bands are invented** (`rules.BANDS`: A < 150 s, B < 180 s, C < 225 s). They have
   never been calibrated against a real run and are almost certainly wrong now that `C = 1`.
   PLAN task 22. **Do this first if Shayne wants to play seriously.**
2. **Run length is unmeasured.** With `C = 1` the whole train takes 361 s to pass the chute
   at 1 m/s, or 85 s at 4.24 m/s. Nobody has played a complete 20-car run end to end.
   A full run may be too long.
3. **Intro physics wording is a DRAFT** for Shayne to replace. It is marked in the file with
   a red bar and `.draft` / `draftlab` styling. Do not polish it as if it were final.
4. **Departures from the brief's own text** are in `SPEC.md` §3.2: 500 kW vs its 2–4.5 MW
   band, 30 m vs its 1 km, 100 000 t/h vs its 1500–3000 t/h. All disclosed to the player in
   the intro. The brief's §3.8 explicitly permits raising the flow rate as one of three ways
   to fix the timescale, so that one is in-bounds; the other two are not and are stated as
   such.
5. **Supplementary check S3** is beyond the brief's four tests. Kept because Test 3 runs at
   `μ → ∞` and so never exercises the adhesion cap. Shayne may strike it.
6. **`PLAN.md` task 5 is superseded** (friction brake removed) and is annotated as such.
   Tasks 22, 17 and 25 are outstanding.
7. **Never written to the repository root.** The root is reserved for the project website and
   is not ours. `git ls-files` must never show a root `index.html`.

---

## 6. Deliberate non-obvious choices

Things that look like bugs but are not:

- **`schedule()` uses rAF *and* a 40 ms timer, mutually exclusive by generation counter.**
  rAF does not fire at all in headless Chrome, which is how the game is screenshot-tested.
  Without the generation counter both fire, both call `frame()`, each schedules another pair,
  and the frame rate doubles every tick until the browser dies. This happened.
- **`dtWall` is clamped at both ends.** The lower clamp matters: rAF timestamps and
  `performance.now()` do not always agree and one negative frame ran the clock backwards.
- **The camera snaps when more than 150 m from target** rather than lerping.
- **The caption positions itself from the HUD's measured height**, because the HUD wraps to
  two lines at phone width.
- **Cosmetic grain particles use their own seeded PRNG stream** that the physics never reads,
  so particle activity cannot perturb a trajectory.
- **`geometry.range()` solves for the candidate cars rather than searching**, so per-step cost
  does not grow with car count.
- **Test 4 derives its own chute-open time** from `m_cap/mdot` rather than hard-coding it,
  which is why it survived the 40× flow change untouched.

---

## 7. Suggested next steps, in order

1. Play a complete run. Nobody has. Measure how long it takes and whether it is enjoyable.
2. Calibrate the grade bands from ~10 real runs (PLAN task 22) and update both `rules.BANDS`
   and `SPEC.md` §5.6 together.
3. Profile on a real phone (task 17). Physics is now only 4 steps/frame, so this should be
   comfortable, but it is unverified on hardware.
4. Cross-browser and GitHub Pages check (task 25), including confirming zero network requests
   in devtools.
5. Ask Shayne to replace the draft physics wording.
6. Only then consider pushing. **Do not push without asking** — nothing has been published yet.

---

## 8. Working agreement with Shayne

- He wrote the brief and is the physicist; when he says the physics is wrong, **check it
  properly and show the numbers** rather than either capitulating or getting defensive. One
  such challenge turned out to be correct behaviour that was simply invisible in play; the
  useful response was measured evidence plus a design fix, not a code change.
- He makes playability calls that override the brief's own parameter bands. Do them, then
  disclose them in `SPEC.md` §3.2 and in the player-facing intro.
- Commit per change with the measurements in the message. The commit log is the real record
  of why every parameter is what it is.
