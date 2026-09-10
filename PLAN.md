# Load the train — build plan

Companion to [SPEC.md](SPEC.md). Section references below are to `SPEC.md` unless marked
"brief".

**No code is written until both documents are reviewed and approved.** Then one task at a
time, in order, committing after each, changing nothing outside the current task.

---

## Ground rules for every task

- One commit per task, message `task NN: <what>`.
- Nothing outside the task's stated scope is touched. No opportunistic refactors.
- **The repository root is never written to.** Only `app/index.html`, `SPEC.md`, `PLAN.md`.
- Any test that fails is reported as a failure and the build stops. No test is weakened, no
  expected value is hard-coded from an observed output, no exception is caught to make a
  failure vanish.
- If a task reveals that `SPEC.md` is wrong, stop and say so rather than diverging quietly.

## Phase map

| Phase | Tasks | Gate to pass before continuing |
|---|---|---|
| A — physics core, headless | 1–9 | **All four brief tests pass to 0.1 %.** |
| B — game rules | 10–13 | Gates and scoring correct against hand-checked cases. |
| C — rendering | 14–17 | 60 fps with 20 cars, readable at 400 px. |
| D — input and UI | 18–21 | Playable start to finish on a phone. |
| E — polish and calibration | 22–26 | Grade bands calibrated; tests still pass. |

Phase A is deliberately front-loaded and entirely headless: the physics is verified before
a single pixel is drawn, so a rendering problem can never be mistaken for a physics problem.

---

## Phase A — physics core, headless

### Task 1 — file skeleton and frozen parameters
Create `app/index.html` with the `<style>` / markup / `<script>` skeleton and a frozen
`PARAMS` object holding every constant from §3, each with its justification as a comment.
Nothing else. No physics, no drawing.

- **Done:** file opens from `file://` with no console errors; `LTT.params()` returns all of
  §3 and the object is frozen.
- **Demonstrated by:** in console, `LTT.params().m_loco === 150000`, and
  `LTT.params().m_loco = 1` throws or silently fails, confirming the freeze.
- **Derived values checked by hand:** train tare 710 t, loaded 2710 t, `F_adhesion`
  441.30 kN, `v_c` 6.798 m/s — all must match §3's derived table.

### Task 2 — momentum-form integrator, constant mass, no forces
RK4 on `(x, p)` per §2.3, with `M` constant and `F_ext = 0`. Fixed step `h = 1/240 s`
simulated. No accumulator or render loop yet; stepping is driven directly by
`LTT.step(dt_sim)`.

- **Done:** from `v_0 = 5 m/s`, `x` advances exactly `5·t` and `v` never changes.
- **Demonstrated by:** `LTT.run({t: 100, throttle: 0, brake: 0, chute: false})` gives
  `x = 500.000` m and `v = 5.000000` m/s to 10 significant figures.

### Task 3 — traction with adhesion limit
Add `F_traction = min(P/v, μ·m_driven·g)` per §2.4, and the accumulator clamp of §4.1.

- **Done:** from rest at tare mass, acceleration is constant at 0.62155 m/s² until
  `v = 6.7981 m/s`, then falls away as `P/v` takes over. No NaN at `t = 0`.
- **Demonstrated by:** **supplementary check S3** (§7.3) against the exact piecewise
  solution: at `t = 60 s`, `v = 21.46690` m/s and `s = 792.8045` m. Closes to 0.1 %.
- **Also demonstrated by:** the crossover lands at `t_c = 10.9374 s`, `s_c = 37.177 m`,
  matching §7.3 by hand.

### Task 4 — adaptive substepping
Bisect a base step while `|Δv| > 0.01·max(v, v_ref)`, to a depth cap, per §4.1. This is
what makes `P/v` tractable near `v = 0`.

- **Done:** a run from `v_0 = 10⁻⁶ m/s` with `μ = Infinity` is stable and accurate, and the
  substep count per base step is reported so the cost is visible.
- **Demonstrated by:** **Test 3** (§7.3), the brief's formulas verbatim: at `t = 10, 30,
  60 s`, `v = 9.19277, 15.92235, 22.51760` m/s and `s = 61.2851, 318.4469, 900.7040` m.
  All closing to 0.1 %. **This is the brief's Test 3 passing.**
- **Watch for:** if the depth cap is hit at the launch, accuracy degrades. The panel reports
  max depth reached; it must stay under the cap.

### Task 5 — brake as a force
Add `F_brake = brake_demand · min(k_b·M·g, μ·M·g)` per §2.5. Never a constant power.

- **Done:** deceleration is 1.1768 m/s² at any mass and any brake-limited speed.
- **Demonstrated by:** stop from 10 m/s in 42.49 m and 8.50 s simulated, at both tare and
  fully loaded mass, confirming mass independence. Matches §3's derived table by hand.
- **Also checked:** brake force never reverses the train through zero; it clamps at `v = 0`.

### Task 6 — car geometry and chute overlap
`geometry` module: car positions from train position, the overlap integral of §2.9, capture
and spill per step. Per-car fill levels with capacity clamping and overflow to ground. No
loss coefficient anywhere.

- **Done:** with the train stationary and the chute over the centre of car 8, 100 % of
  released grain is captured by car 8 and nothing else. With the chute over a coupling gap,
  0 % is captured. Straddling a gap edge, capture is the linear overlap fraction.
- **Demonstrated by:** a table of chute positions swept across one 17.0 m pitch in 0.1 m
  steps, with captured fraction printed. It must be 1.0 across the 15.5 m trough (less the
  half-aperture at each end), ramp linearly over `w = 0.8 m` at each edge, and be 0.0 across
  the 1.5 m gap centre. Integrated capture over one full pitch must equal
  `1 − 1.5/17.0 = 91.18 %`.
- **Also checked:** loco roof captures nothing; ahead of car 1 and beyond car 20 capture
  nothing; a car at capacity captures nothing and all of it goes to ground.

### Task 7 — mass ledger, and Test 1
`ledger` module: `m_released`, `m_cars`, `m_ground` per §2.8, accumulated inside the physics
step only. Written from §2.9 directly, not from `geometry`'s internals.

- **Done:** `m_released = m_cars + m_ground` holds at **every** physics step, not just at the
  end (possible because landing is instantaneous, decision 5).
- **Demonstrated by:** **Test 1** (§7.1) — the scripted 400 s simulated sequence that opens
  the chute over gaps, over the loco, past car 20, and over a full car. All three quantities
  printed; closure to 0.1 %. The harness asserts the identity every step. **This is the
  brief's Test 1 passing.**

### Task 8 — energy ledger, and Tests 2 and 4
`ledger` gains `W_traction`, `W_brake`, `E_diss` and `J_acc` per §2.8. **`E_diss` is coded
from §2.7 route 2 — the inelastic-collision argument — not from the integrator's algebra.**
`ΔKE` evaluated directly as `½Mv²`, never accumulated.

- **Done:** `W_traction − W_brake = ΔKE_total + E_diss` closes to 0.1 % on a run with
  throttle, brake and large spill all exercised.
- **Demonstrated by:** **Test 2** (§7.2) on the Task 7 sequence, every term reported
  separately with the residual as an absolute value and as a fraction of `W_traction`.
  **This is the brief's Test 2 passing.**
- **And by:** **Test 4** (§7.4) — coast from `v_0 = 2 m/s`, `M_0 = 710 t`, chute open 60 s
  simulated. `M_final = 751.6667 t`, `v_final = 1.889135 m/s`,
  `KE_final/KE_0 = 0.94456763`. **This is the brief's Test 4 passing.**
- **Two independent cross-checks, both required:** `J_acc = 80 979.75 N·s` against
  `p·ln(M_f/M_0)`; and `ΔKE + E_diss = 0` under coasting, where the predicted magnitudes are
  ∓78 713.97 J. A factor-of-two error surfaces here as a factor-of-two residual.
- **Risk, called out in the brief:** this is the single most likely physics error in the
  build. `E_diss` must come out `½v²dm` and not `v²dm`. If the residual is a clean factor of
  two, that is the bug, and §2.7 route 2 is the arbiter.

### Task 9 — test panel and console API
`tests` and `LTT` per §7.5. Panel at `app/index.html?test` and behind a DEBUG button:
button per test, RUN ALL, and a table of computed / predicted / difference / relative error
with PASS-FAIL per row. Test 3 also prints the naive-versus-exact comparison of §7.3.

- **Done:** RUN ALL reports 4 of 4 brief tests plus S3 passing, from a cold load, in both
  Chrome and Safari, and from `file://` as well as over HTTP.
- **Demonstrated by:** `LTT.runTest(3, {mu: Infinity, P: 2e6, M: 5e5, t: 30})` accepts
  arbitrary overrides and prints predictions Shayne can check against his own arithmetic.
- **Also checked:** determinism — the same scripted sequence run twice gives bit-identical
  ledgers (§4.2).

> ### GATE A — do not start Phase B until this holds
> All four of the brief's tests pass to 0.1 %, plus S3, plus both Test 4 cross-checks, from a
> cold load, headless, with no rendering code in the file. Report the actual numbers.

---

## Phase B — game rules

### Task 10 — run state machine
Title → intro → running → paused → report, plus practice mode. Clock starts on first
non-neutral control input or first chute opening (§5.4). Pause freezes both clocks and makes
controls inert.

- **Done:** every transition reachable and reversible where §5.5 says it is; the clock does
  not advance while paused or before the first input.
- **Demonstrated by:** scripted transition walk through all states; elapsed time is 0.000
  until the first input, and unchanged across a 10 s pause.

### Task 11 — gates and scoring
The two gates of §5.6, evaluated at DONE. `loss = m_ground / m_released` (decision 14).

- **Done:** gates read the ledger's own quantities, so Gate 2 and Test 1 use the same two
  numbers.
- **Demonstrated by:** three hand-constructed end states — all cars at 95.0 % exactly with
  loss 0.999 % (PASS, both at the boundary); one car at 94.9 % (FAIL, that car named); loss
  at 1.001 % (FAIL, figure quoted). Boundary behaviour must be `≥ 95.0` and `< 1.00`
  exactly as written.

### Task 12 — free shunting and track limits
Reverse throttle, reverser interlock below 0.2 m/s, track from −1100 m to +700 m with soft
limits drawn and no crash state (decision 11, §3).

- **Done:** the train can be driven back and forth over the full track, any car can be
  brought under the chute, and the reverser refuses to change above 0.2 m/s.
- **Demonstrated by:** scripted run that fills car 20, reverses, and tops up car 3, with the
  ledger still closing Tests 1 and 2 to 0.1 % across the direction reversal.
- **Watch for:** sign errors in `W_traction` and `W_brake` under reverse. Both work integrals
  must remain positive quantities; `v` and `F` change sign together.

### Task 13 — overflow and spill accounting
Overflow past capacity goes to ground (§2.9). Spill heightfield in 2 m bins for later
drawing.

- **Done:** holding an open chute over a full car adds nothing to `m_cars` and everything to
  `m_ground`; the heightfield's total equals `m_ground`.
- **Demonstrated by:** fill car 5 to capacity, hold 60 s more simulated, then check
  `m_cars` unchanged, `m_ground` risen by 41.667 t, Test 1 still closing, and the
  heightfield sum matching `m_ground` to 0.1 %.

---

## Phase C — rendering

### Task 14 — canvas, camera, and the play view
Single canvas with `devicePixelRatio` scaling. Camera locked with the chute at ~55 % of
width, ~12 px/m at 400 px (§5.1). Ground, rails, sky gradient, chute tower.

- **Done:** about 33 m of track visible at 400 px; camera never jitters as the train moves.
- **Demonstrated by:** screenshot at 400 × 800 and at 1440 × 900; the chute stays put and
  the scale bar reads 33 m and ~120 m respectively.

### Task 15 — train art
Loco and 20 cars in the warm storybook palette of §5.7: thick `#3B2A1F` outlines, rounded
forms, one soft gradient per fill. Grain level drawn from the fill fraction via the bulk
density of §3.

- **Done:** loco and cars readable at 400 px width; coupling gaps visibly gaps; fill level
  legible per car.
- **Demonstrated by:** side-by-side screenshots of a car at 0 %, 50 %, 95 % and 100 %, at
  both phone and desktop scale, with the 95 % case distinguishable from 100 % at a glance.

### Task 16 — grain stream and spill piles
Cosmetic particles from a fixed pool capped at 400, seeded PRNG in a stream the physics
never reads (§4.2). Spill mounds from the Task 13 heightfield.

- **Done:** the stream visibly lands in a trough or misses onto the ground, matching what
  `geometry` computed; mounds grow where grain was actually lost.
- **Demonstrated by:** deliberately crossing a gap with the chute open — the visible miss and
  the `m_ground` increment must occur at the same moment and in the same place.
- **Also checked:** particle pool never grows; determinism of the physics is unaffected by
  the particle stream (re-run Task 9's determinism check).

### Task 17 — performance
Profile with all 20 cars loaded and grain flowing.

- **Done:** 60 fps sustained on a mid-range phone (brief §2), with 100 physics steps per
  frame at C = 25.
- **Demonstrated by:** frame-time histogram over 60 s of play, 99th percentile under 16.7 ms;
  physics time per frame reported separately from render time.
- **If it misses:** the lever is the base step, and any change to it must be re-justified in
  §4.1 and Phase A re-run. Render cost is cut first — physics accuracy is not the lever.

---

## Phase D — input and UI

### Task 18 — controls
Continuous sticky slider, reverse toggle, chute toggle, full keyboard set (§5.3). Sampled
once per frame and held across substeps (§4.2).

- **Done:** works on touch and with mouse and keyboard; slider holds where released; thumb
  does not cover the chute in portrait; left-handed mirror option present.
- **Demonstrated by:** a full run completed on a phone using touch only, and a second run
  using keyboard only.
- **Also checked:** determinism holds with the frame-sampled control path (§4.2).

### Task 19 — HUD and ledger panel
Speed, both clocks, mass, loss against the 1 % budget, and the always-visible
`v·dM/dt` accretion bar. One tap expands the full ledger with live Test 1 and Test 2 closures
(§5.2).

- **Done:** every §5.2 item present; the accretion bar responds visibly to loading while
  moving and reads zero when stationary or when the chute is shut.
- **Demonstrated by:** load at 0.1 m/s and at 2 m/s and compare the bar against
  `v·dM/dt` computed by hand from the flow rate — 69 N and 1389 N respectively at full
  capture.

### Task 20 — fill strip
All 20 cars at a glance, 95 % line marked, current car highlighted, gaps drawn as gaps. Top
in portrait, side in landscape (§5.1).

- **Done:** the whole train's fill state is judgeable without moving the camera; the caption
  reads `CAR n` / `GAP n–n+1` / `LOCO` / `NO CAR` correctly.
- **Demonstrated by:** a state with cars 1–10 at 96 %, car 11 at 94 %, rest empty — car 11
  must be identifiable as the under-95 % one at 400 px width without zooming.

### Task 21 — intro popups and end-of-run report
Four skippable, re-openable cards (§5.8) and the report of §5.4.

- **Done:** intro states the zero-resistance idealisation **and** that the train never slows
  on its own; states C = 25; explains the accretion retardation without the words "rocket
  equation"; states momentum conservation **only while coasting**. Draft physics copy wrapped
  in a `<!-- DRAFT COPY — REPLACE -->` block.
- **Demonstrated by:** `grep -ci "rocket" app/index.html` returns 0; grep for "momentum"
  shows every occurrence carries the coasting caveat; the draft block is present and
  delimited.
- **Report checked by:** a failing run naming the specific cars, and the full ledger with
  both closures shown.

---

## Phase E — polish and calibration

### Task 22 — grade band calibration
Play ten complete runs. Set the A/B/C bands from the distribution, replacing the provisional
values in §5.6 (A < 2:30, B < 3:00, C < 3:45).

- **Done:** bands set from measured times, and both `SPEC.md` §5.6 and the code updated
  together.
- **Demonstrated by:** the ten run times tabulated in the commit message, with the chosen
  bands and the reasoning. A must be achievable but not routine.

### Task 23 — practice mode
No clock, no gate, free reset, jump-to-any-car, ledger expanded by default (§5.5).

- **Done:** practice never writes a best time and never shows a gate verdict.
- **Demonstrated by:** enter practice, jump to car 15, load, reset, and confirm
  `localStorage` is untouched.

### Task 24 — persistence
Best passing time and the intro-seen flag only, both resettable, both wrapped so a throwing
or empty `localStorage` is harmless (§5.5).

- **Done:** best time survives reload; the game runs normally with `localStorage` disabled.
- **Demonstrated by:** a run in a private window with site data blocked — no console errors,
  no missing UI.

### Task 25 — cross-platform and deployment check
`file://` and GitHub Pages; Chrome, Safari, Firefox; portrait and landscape phone, tablet,
desktop.

- **Done:** no console errors, no external requests, correct layout at every size.
- **Demonstrated by:** the browser devtools network panel showing **zero** requests beyond
  the document itself, on both `file://` and Pages.
- **Also checked:** the repository root is untouched — `git log` shows no commit that adds or
  modifies `index.html` at the root.

### Task 26 — final test run and sign-off
Re-run everything from Phase A on the finished file.

- **Done:** all four brief tests plus S3 pass to 0.1 % on the shipped file, with rendering,
  input and UI all present.
- **Demonstrated by:** the RUN ALL panel output pasted into the final commit message, with
  actual numbers, not a claim of success.

---

## Risk register

| Risk | Where it bites | Mitigation |
|---|---|---|
| **Factor of two in `E_diss`** — the brief names this as the most likely error | Task 8, Test 2 | `E_diss` coded from §2.7 route 2, independently of the integrator. A residual that is a clean 2× or 0.5× identifies it immediately. Test 4's two cross-checks catch it a second time. |
| Sign errors under reverse | Task 12 | Explicit reversal test with both work integrals checked positive. |
| Test 3's `v → 0` singularity | Task 4 | Adaptive substepping in the core integrator, plus the stated `v_0 = 10⁻⁶ m/s`. Substep depth reported so a silent cap-hit cannot pass unnoticed. |
| 100 physics steps per frame too slow | Task 17 | Measured, not assumed. Render cost is cut first; the base step is the last resort and any change re-runs Phase A. |
| Readability at 400 px | Tasks 15, 20 | Thick outlines throughout, and an explicit 94 %-versus-96 % discrimination test at phone width. |
| Adaptive substepping breaks determinism | Tasks 4, 18 | Substep decisions depend only on state and `h`, never on wall time. Determinism re-checked at Task 9 and again at Task 18. |
| Scope creep into the repository root | all | Task 25 checks `git log` for root changes; the ground rules forbid it. |

---

## What is not in this plan

Per §8: no sound, no server leaderboard, no coupling slack, no grade or curvature, no
rolling resistance or drag, no network requests, and **nothing at the repository root**.
