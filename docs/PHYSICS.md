# Physics notes — the things that must not be got wrong twice

Companion to `../HANDOVER.md`. `SPEC.md` §2 is the full derivation; this is the short list of
facts that are load-bearing, easy to "helpfully" break, and expensive to rediscover.

---

## 1. The five facts

- **The integrator state is momentum**, `p = Mv`, not velocity. With grain arriving at
  `u = 0` the variable-mass law is just `dp/dt = F_ext`, so the `v·dM/dt` retarding term
  **is never written down anywhere** — it emerges when `v` is recovered as `p/M`. Do not
  "add" it. If you find yourself typing `v*dMdt` into the integrator, something has gone
  wrong.
- **The mass update happens after the mechanical update, deliberately.** That operator split
  is the perfectly-inelastic-collision picture: mass rises with `p` untouched, so `v` drops
  and exactly `½v²dm` of kinetic energy disappears. That is where the brief's factor of two
  lives.
- **`ledger` must not be written from `physics`'s algebra.** It is handed raw per-substep
  samples and derives everything from `SPEC.md` §2.7 route 2 on its own. This separation is
  the brief's explicit safeguard against the factor-of-two error, and is a review criterion,
  not a style preference.
- **There is no loss coefficient.** Loss is the aperture–trough overlap integral and nothing
  else. Integrated over one pitch it is exactly `L_body/L_pitch` = 91.176471 %.
- **There is no brake.** `braking()` does not exist. Retardation is `traction()` with a
  negative argument.

---

## 2. Current parameters

```
g 9.80665     m_loco 150 t    m_driven 150 t   P_rated 1 MW     mu 0.30
N_cars 20     m_tare 28 t     m_cap 100 t      L_body 15.5 m    L_pitch 17.0 m
mdot 27 777.8 kg/s (100 000 t/h)               w_chute 0.8 m    L_loco 21 m
x_start -30   x_min -60       x_max 700        C_time 1         h_step 1/240 s
fill_target 0.90              loss_budget 0.01                  test_tol 0.001
```

Derived: `M_tare` 710 t, `M_full` 2710 t, `F_adhesion` 441.30 kN, `v_cross` 2.266 m/s,
`train_len` 361 m, `t_fill_car` 3.60 s, `t_fill_all` 72.0 s, `gap_duty` 8.82 %.

Everything derived is computed in `DERIVED` from `PARAMS`; nothing re-derives a constant by
hand. If you change a parameter, the code follows — the **documents** are what go stale.

---

## 3. Traction, and the bug that shaped the controls

```
F = throttle · min( P_rated / |v| , μ · m_driven · g )      throttle ∈ [−1, +1]
```

**The throttle scales the whole `min()`.** It used to scale only the power term —
`min(throttle·P/v, μmg)` — and below `throttle · v_c` the `min()` then always selected the
adhesion cap, so **1 % throttle and 100 % throttle both produced the full 441.30 kN**. The
throttle had no authority at all from rest. That single sign-of-the-brackets error was the
cause of "control fidelity is way too coarse", and fixing it is what made one signed throttle
a workable control scheme.

---

## 4. The two numbers that shape the game

```
loading terminal speed   v_eq = sqrt(P_rated / mdot) = 6.000 m/s
one-pass fill limit      L_body / (m_cap/mdot)       = 4.306 m/s
```

**At 1 MW the first is above the second, deliberately.** Until 2026-09-11, at 500 kW, it was
the other way round (4.243 vs 4.306): the grain held the train just *below* the speed at
which a car could still be filled in one pass, so holding full throttle from the start line
to the end of the train was very nearly a winning strategy on its own — measured 94.9 %
aboard. Shayne asked for 1 MW; the knee is now crossed, full throttle settles at 6.24 m/s and
loads 69.7 %, and the player must hold the train in the band themselves.

The settled speed sits a little above `v_eq` because grain is captured only over the car
bodies — 91.18 % duty — so the effective flow is `0.9118·mdot`, giving 6.28 m/s.

**If you retune `P_rated` or `mdot`, recompute both numbers and say which side of the knee
you are on.** It changes what the game is about.

### The speed the pass gate actually sets

4.306 m/s is the speed at which a car fills to the *brim*. What the player is scored against
is `fill_target`, so the speed that matters is

```
v_max = L_body / (fill_target · m_cap / mdot)   = 4.784 m/s at 90 %   (4.532 at 95 %)
```

Measured values in `MEASUREMENTS.md` §2.

### The loss budget is what makes it a game

Gap duty is **8.82 %** of track length against a **1 %** loss budget. Holding the chute open
for a whole pass therefore fails on loss alone, whatever the fill. The chute must be shut
over all nineteen coupling gaps — a 0.33 s window each at 4.5 m/s.

---

## 5. Stopping distance is piecewise — corrected 2026-09-12

The same `min()` that caps tractive effort caps retarding effort, so:

```
v ≤ v_c :  s_stop = M·v² / (2·F_a)
v > v_c :  s_stop = M·(v³ − v_c³) / (3·P_rated)  +  M·v_c² / (2·F_a)
```

`rules.stopDistance()` used only the constant-power branch at every speed, with a comment
claiming that below `v_c` adhesion gives *more* force than `P/|v|` so the estimate was
conservative. **It is the other way round** — `min()` selects the adhesion cap there, which is
*less* force, so the train takes *longer* to stop and the old form under-estimated. Bounded by
`M·v_c²/(6·F_a)` = 5.26 m loaded, 1.38 m tare; the trip stop's flat 6 m margin covered it, so
nothing ever overran, but it was safe by accident and the raise to 1 MW had doubled the error
(`v_c` went 1.133 → 2.266 m/s and the error goes as `v_c²`).

Measured table in `MEASUREMENTS.md` §3.

---

## 6. Deliberate non-obvious choices

Things that look like bugs and are not:

- **`schedule()` uses rAF *and* a 40 ms timer, mutually exclusive by generation counter.**
  rAF does not fire at all in headless Chrome, which is how the game is screenshot-tested.
  Without the counter both fire, both call `frame()`, each schedules another pair, and the
  frame rate doubles every tick until the browser dies. This happened.
- **`dtWall` is clamped at both ends.** The lower clamp matters: rAF timestamps and
  `performance.now()` do not always agree, and one negative frame ran the clock backwards.
- **The camera snaps when more than 150 m from target** rather than lerping.
- **The caption positions itself from the HUD's measured height**, because the HUD wraps to
  two lines at phone width.
- **Cosmetic grain particles use their own seeded PRNG stream** that the physics never reads,
  so particle activity cannot perturb a trajectory.
- **`geometry.range()` solves for the candidate cars rather than searching**, so per-step cost
  does not grow with car count.
- **Test 4 derives its own chute-open time** from `m_cap/mdot` rather than hard-coding it,
  which is why it survived the 40× flow change untouched.
