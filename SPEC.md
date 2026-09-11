# Load the train — specification

Project 1, Basic Skills for Experimentalists (TIGP 2026). Author of the brief: Shayne
Bennetts. This specification written by Claude from `loadthetrainspec.md` and three rounds
of interview; **not yet reviewed**.

Status: awaiting review. No code has been written.

---

## 0. Decisions taken in interview

The brief left these open. Answers given by Shayne, recorded here because they are load
bearing for everything below.

| # | Question | Decision |
|---|---|---|
| 1 | Timescale (§3.8 of brief) | Fully realistic parameters, **explicit time compression factor C = 25**. 1 s wall clock = 25 s simulated. |
| 2 | Camera / overview (§5) | Close play view locked near the chute, **plus a persistent fill strip** showing all 20 cars. |
| 3 | Scoring (§5) | **Hard gate then time.** Every car ≥ fill_target AND loss < 1 % is pass/fail; passing runs ranked by elapsed time. The target was the brief's 95 % until 2026-09-11 and is now **90 %** — see §9 item 7. |
| 4 | Run structure | **Free shunting**, forward and reverse throughout; run ends when the player presses DONE. |
| 5 | Grain flight | **Instantaneous landing.** Grain accretes at the release instant, at the chute's x. The falling column is cosmetic. Mass ledger stays at three buckets. |
| 6 | Car opening | **Full-length open trough**, 15.5 m open on a 17.0 m coupler pitch. |
| 7 | Test harness | **Panel plus console API**, both inside `app/index.html`. |
| 8 | Art direction | **Warm storybook** cartoon, drawn procedurally in Canvas 2D. No image files. |
| 9 | Controls | **Continuous slider**, full brake to full power, with a reverse toggle. |
| 10 | Physics HUD | **Minimal HUD plus one-tap ledger panel.** Accretion drag `v·dM/dt` always visible. |
| 11 | Track layout | **Loco leads, open track.** No buffer stops, no crash state. |
| 12 | Modes | **Timed run plus practice mode**, personal best in `localStorage`. |
| 13 | Test 3 | Run with **μ → ∞** so the adhesion limit never binds and the brief's closed forms hold verbatim. See §7.3. |
| 14 | Loss budget denominator | **loss = grain on ground / grain released.** The same two quantities Test 1 closes on. |

---

## 1. What it does

A locomotive at the head of twenty open-top grain hopper cars starts 1 km short of a fixed
loading chute. The player drives the train under the chute and opens the chute to load each
car in turn. The run is a pass only if every car finishes at **90 %** of capacity or more
(the brief says 95 %; see §9 item 7) and total grain loss stays under 1 % of
grain released. Passing runs are ranked by elapsed time.

The tension is physical, not scripted. Filling a car takes 144 s of simulated time while
the car spans only 15.5 m of track, so the train must be nearly stationary to fill. Loading
on the move costs twice: grain released over a coupling gap falls on the ground, and the
arriving grain exerts a real retarding force `v·dM/dt` on the train that the player can
watch on the HUD.

Single self-contained file, `app/index.html`. Plain HTML, CSS and JavaScript. No build step,
no dependencies, no network loads. Works opened from `file://` and served from GitHub Pages.

---

## 2. Physical model

### 2.1 Idealisations, and their disclosure

**Zero rolling resistance. Zero aerodynamic drag.** Deliberate, and stated to the player in
the intro with its consequence spelled out: *the train never slows down on its own, so
every stop must be braked.*

Further idealisations, also disclosed:

- The train is a **single rigid body**. No coupling slack, no buffer compression, no slack
  action. Position and velocity are scalars shared by loco and all cars.
- Grain lands **at the instant of release**, at the chute's horizontal position (decision 5).
- Grain is treated as a **continuum** in the physics. The visible particles are cosmetic.
- Track is level and straight. No grade, no curvature.

### 2.2 This is mass accretion

The general variable-mass law is

```
F_ext = d(Mv)/dt − u·(dM/dt)
```

with `u` the velocity of the mass being added or removed.

Grain leaves the chute with **zero horizontal velocity**, so `u = 0` and that term
vanishes. What remains **retards** the train. A rocket ejects mass at a relative exhaust
velocity and that term produces thrust; this is the opposite sign. The textbook analogue is
sand falling onto a moving conveyor belt, or a hopper car loaded in motion.

*The phrase "rocket equation" appears nowhere in the code, the UI, or the intro text.*

### 2.3 Equation of motion

With `u = 0`:

```
F_traction − F_brake = d(Mv)/dt = M·(dv/dt) + v·(dM/dt)
```

Rearranged into the form the integrator actually uses, with `p ≡ Mv`:

```
dp/dt = F_traction − F_brake        (momentum form; u = 0)
dx/dt = p / M
```

`M(t) = m_loco + N·m_tare + m_grain_loaded(t)`, the instantaneous total moving mass.

The retarding term is **not written anywhere in the integrator**. It emerges: the code
advances `p` by the external impulse and advances `M` by the captured grain, then recovers
`v = p/M`. Expanding that recovery reproduces `M·dv/dt = F_ext − v·dM/dt` identically. There
is no tuned penalty constant anywhere in the build, and §7 verifies this against closed
forms that were derived independently of the code.

**`dM/dt` is the *captured* flow rate, not the released flow rate.** Grain that misses a car,
or overflows a full one, never joins `M` and never appears in the momentum or energy balance.

### 2.4 Traction and the adhesion limit

```
F_traction = throttle · min( P_rated / v , μ · m_driven · g ) · direction
```

**The throttle commands a fraction of the available tractive effort**, which is what a
locomotive controller actually does: at low speed the notch sets tractive effort against a
current limit, and only at higher speed does the power rating become binding. Adhesion caps
the maximum either way.

**Corrected during the control-fidelity pass.** This document originally had the throttle
scaling only the power term, `min(throttle·P/v, μ·m_driven·g)`. That is wrong, and it made the
game nearly unplayable: below `throttle · v_c` the `min()` always selected the adhesion cap,
so **1 % throttle and 100 % throttle both produced the full 441.30 kN**. The throttle had no
authority whatsoever from rest. Measured on the corrected law, from rest at tare mass, over
one second of wall clock (25 s simulated):

| notch | `F_traction` | `a` | reached | travelled |
|---|---|---|---|---|
The notch scales the whole `min()`, so 2 % of full effort is 2 % of full effort from rest,
at any speed. At 1 MW, `v_c = 2.266 m/s`: adhesion governs the launch, constant power
governs the rest. Full throttle from rest covers 0.31 m in the first second — unchanged,
because the first second is adhesion-limited and the power rating does not enter it — and
the 30 m approach in 10.5 s, arriving at the chute at 4.95 m/s.

The adhesion cap is still mandatory: `P/v` diverges as `v → 0`, which without it produces
either an instantaneous launch or a NaN at `t = 0`. With the parameters of §3 it binds below

```
v_c = P_rated / (μ · m_driven · g) = 1 MW / 441.30 kN = 2.266 m/s
```

so adhesion governs the launch and constant power governs everything above a walking pace.

### 2.5 Retardation — regenerative, not friction

**There is no friction brake.** The locomotive has one bipolar control. Pushing it to the
opposite sign while rolling puts the traction motors into generation: same machine, same
rating, same adhesion cap, opposite direction. So retardation is not a separate model at
all — it is `traction()` with a negative argument, and the code has no brake function.

```
F = throttle · min( P_rated / |v| , μ · m_driven · g )        throttle ∈ [−1, +1]
```

This still meets the brief's two requirements on braking. It is a **force**, not a constant
power; and it is **adhesion-limited**, capped at 441.30 kN, so it does not diverge as
`v → 0` the way a naive constant-power brake would. The `min()` is what guarantees that.

**The consequence the player feels.** The stopping distance is **piecewise**, because the
same `min()` that caps tractive effort caps retarding effort: adhesion binds below `v_c`,
constant power above it.

```
v ≤ v_c :  s_stop = M·v² / (2·F_a)
v > v_c :  s_stop = M·(v³ − v_c³) / (3·P_rated)  +  M·v_c² / (2·F_a)
```

**This document previously gave only the constant-power branch**, `M·|v|³/(3·P_rated)`, at
every speed, and `rules.stopDistance()` implemented it that way with a comment claiming it
was conservative. It is not: below `v_c` the `min()` selects the adhesion cap, which is
*less* force than `P/|v|`, so the train takes *longer* to stop and the constant-power form
under-estimates — by up to `M·v_c²/(6·F_a)` = 5.26 m loaded, 1.38 m tare. The trip stop's
flat 6 m margin covered it, so nothing ever failed, but it was safe by accident and the
raise to 1 MW had doubled the error. Corrected 2026-09-12; the piecewise form agrees with
the integrator to 0.01 m at every speed and mass below.

Still strongly mass dependent, unlike the friction brake it replaced — **measured**, by
driving the shipped integrator at full reverse effort until `v` reaches zero:

| speed | empty, 710 t | loaded, 2710 t |
|---|---|---|
| 0.5 m/s | 0.20 m | 0.77 m |
| 1.0 m/s | 0.80 m | 3.07 m |
| 2.0 m/s | 3.22 m | 12.28 m |
| 4.0 m/s | 16.52 m | 63.07 m |
| 6.0 m/s | 52.50 m | 200.38 m |

At loading speed — 4.3 m/s — a loaded train needs about 77 m to stop, four and a half car
pitches. With no friction brake to fall back on, creeping is not optional, which suits a
game whose whole action is placing a 15.5 m trough under a 0.8 m chute.

Holding reverse effort through `v = 0` correctly drives the train backwards; there is no
clamp, because reversing is what that control does. Effort is continuous through zero, so
the integrator needs no special case.

Energy is reported as the signed integral of `F·v`, split by sign: `W_traction` where the
motors drive, `W_regen` where they generate and give energy back. Never as a power figure.

### 2.6 Momentum

`Mv` is conserved **only while coasting**, meaning traction and brake force both exactly
zero. Under throttle or brake the rails supply an external horizontal force and
`d(Mv)/dt = F_ext`.

*No player-facing text claims general momentum conservation.* The coasting caveat is stated
wherever the conservation is mentioned.

### 2.7 Energy, and the factor of two

The brief warns that this is the most likely physics error in the build, and that the
reasoning which writes the integrator must not also write the ledger. So the factor is
derived here **twice, by two independent routes**, before any code exists. The integrator
(§2.3) uses the momentum form; the ledger (§2.8) accumulates the four terms below
separately and the identity is then *checked*, not assumed.

**Route 1 — differentiate the total kinetic energy.**

Total KE of the moving system, `K = ½Mv²`:

```
dK/dt = ½v²·(dM/dt) + Mv·(dv/dt)
```

From the momentum form, `M·(dv/dt) = F_ext − v·(dM/dt)`. Substituting:

```
dK/dt = ½v²·(dM/dt) + v·[ F_ext − v·(dM/dt) ]
      = F_ext·v + ½v²·(dM/dt) − v²·(dM/dt)
      = F_ext·v − ½v²·(dM/dt)
```

Hence

```
∫ F_ext·v dt = ΔK + ½ ∫ v²·(dM/dt) dt
```

**Route 2 — treat each parcel as a perfectly inelastic collision.**

Over `dt`, a parcel `dm` at rest merges with mass `M` moving at `v`. Momentum before and
after (the external impulse contributes at `O(dt²)` here and drops out):

```
Mv = (M + dm)·v'        ⟹    v' = Mv/(M + dm)
```

Kinetic energy after:

```
½(M + dm)v'² = ½ M²v² / (M + dm) = ½Mv²·(1 − dm/M + O(dm²))
             = ½Mv² − ½v²·dm + O(dm²)
```

So exactly `½v²·dm` of kinetic energy disappears per parcel — the heat of the inelastic
transfer. Independently, the parcel itself gains `½·dm·v'² → ½v²·dm` of kinetic energy. The
train must therefore supply `½v²dm + ½v²dm = v²dm`, which is precisely the work done against
the retarding force `v·(dM/dt)` at speed `v`, namely `v²·(dM/dt)`. **Three quantities, all
mutually consistent, factor of two confirmed by both routes.**

| Term | Rate |
|---|---|
| Work absorbed from the train | `v²·(dM/dt)` |
| Kinetic energy gained by the grain | `½v²·(dM/dt)` |
| Dissipated as heat | `½v²·(dM/dt)` |

Loaded grain then rides with the train, so its kinetic energy is already inside `½Mv²`
(because `M` includes loaded grain). **Only the dissipated half is a separate ledger line.**
Grain that misses a car is never accelerated and enters neither energy term.

### 2.8 The ledger

Six quantities accumulated inside the physics step, never in the render step, each by its own
independent accumulation:

| Symbol | Accumulated as |
|---|---|
| `W_traction` | `Σ max(F·v, 0) · h`, the energy the motors put in |
| `W_regen` | `−Σ min(F·v, 0) · h`, the energy the motors took back |
| `E_diss` | `Σ ½ v² · dm_captured` |
| `KE(t)` | evaluated directly as `½ M v²`, not accumulated |
| `m_released` | `Σ ṁ_flow · h` while the chute is open |
| `m_ground` | `Σ (dm_released − dm_captured)` |

`m_cars` is the sum of the twenty per-car fill levels, held independently of `m_released`
and `m_ground`.

Also accumulated, for the Test 4 cross-check of §7.4: the accretion impulse
`J_acc = Σ v · dm_captured`.

### 2.9 Grain loss is geometric

No loss coefficient exists in the build. Loss falls out of geometry alone.

Let the chute aperture span `[x_c − w/2, x_c + w/2]`. Each car `i` presents an open trough
spanning `[X_i, X_i + L_body]` in track coordinates, where `X_i` follows rigidly from the
train position. Then per step, with `dm_released = ṁ_flow · h`:

```
overlap_i  = max(0, min(x_c + w/2, X_i + L_body) − max(x_c − w/2, X_i))
capture_i  = dm_released · overlap_i / w
```

Everything not captured by some car is on the ground: coupling gaps (1.5 m of every 17.0 m
pitch, 8.82 %), the locomotive roof, the space ahead of car 1, and the space beyond car 20.

**Overflow.** If `capture_i` would take car `i` past capacity, only the remainder is
accepted and the excess goes to `m_ground`. So a car held under an open chute past 100 %
spills, and that spill counts against the 1 % budget.

Aperture straddling a coupling gap is handled correctly by construction: the overlaps of the
two adjacent cars sum to less than `w`, and the shortfall is the loss.

**Grain on the locomotive roof.** The loco leads (decision 11), so it passes beneath the
chute first. Grain released then lands on the loco and is lost. Deliberate, instructive, and
purely geometric — the loco simply presents no capturing trough.

---

## 3. Parameters, with justification

Every value, and why. `g = 9.80665 m/s²` exactly.

| Quantity | Value | Justification |
|---|---|---|
| Locomotive mass `m_loco` | **150 t** | Mid of the brief's 120–200 t. A six-axle Co-Co road freight unit. |
| Driven mass `m_driven` | **150 t** | All six axles driven, so the full loco mass sits on driven axles. Cars are unpowered. |
| Rated power `P_rated` | **1.0 MW** | Still below the brief's 2–4.5 MW band, but a realistic rating for a large yard shunter. See §3.2. **Deliberately set above the accretion-drag knee** — see §3.2 and §9 item 6. |
| Adhesion coefficient `μ` | **0.30** | Mid of the brief's 0.25–0.35 for dry rail with sanding. Gives `F_adhesion = 441.30 kN`. |
| Car tare `m_tare` | **28 t** | Mid of the brief's 25–30 t. |
| Car capacity `m_cap` | **100 t** | Top of the brief's 60–100 t, chosen so the 20 × 100 t = 2000 t figure matches the brief's own timescale arithmetic. |
| Number of cars `N` | **20** | Given. |
| Chute flow `ṁ_flow` | **100 000 t/h** = 27 777.8 kg/s | **40× the brief's 1500–3000 t/h band. See §3.2.** At a real rate the accretion term is unobservable; see below. |
| Chute aperture `w` | **0.8 m** | A single-spout terminal loading chute. Narrow relative to the 1.5 m gap, so a gap cannot be straddled harmlessly. |
| Chute lip height | **4.0 m** above car rim | Visual only; instantaneous landing (decision 5). |
| Car body length `L_body` | **15.5 m** | Open trough length of a large covered hopper. |
| Coupler pitch `L_pitch` | **17.0 m** | Gives gap `= 1.5 m`, so gap duty `= 8.82 %` of track length. |
| Loco length | **21.0 m** | Typical Co-Co over couplers. |
| Grain bulk density | **0.77 t/m³** | Wheat. Sets the visual fill height: 100 t → 129.9 m³, against a 15.5 × 3.0 × 2.8 m interior = 130.2 m³. Consistent to 0.3 %. |
| Approach distance | **30 m** | **Departs from the brief's 1 km. See §3.2.** |
| Track extent | **−60 m to +700 m** | Chute at `x = 0`, loco nose starts at `x = −30`. Train is 361 m long. Car 20's trough reaches the chute with the nose at `+345.5 m`, and its centre sits under the chute at `+353.25 m` (corrected at task 6 from an earlier estimate of +340 m). ~347 m of spare beyond, ample for free shunting. No buffer stops. |
| Pass threshold `fill_target` | **90 %** | **Departs from the brief's 95 %. See §3.2 and §9 item 7.** |
| **Time compression `C`** | **1** — none | The clock is real. See §3.1. |

Derived, for reference:

| Quantity | Value |
|---|---|
| Train tare mass | 710 t |
| Train fully loaded | 2710 t |
| Adhesion force | 441.30 kN |
| Adhesion/constant-power crossover `v_c` | 2.266 m/s |
| Launch acceleration, tare | 0.6216 m/s² |
| Launch acceleration, loaded | 0.1628 m/s² |
| Tractive effort at 1 m/s (full) | 441.3 kN (adhesion-capped; `P/v` would be 1000 kN) |
| Stopping distance from 1 m/s, tare / loaded | 0.80 m / 3.07 m (adhesion-limited below `v_c`) |
| Time to fill one car | 3.60 s (3.24 s to the 90 % gate) |
| Time to fill all twenty | 72.0 s |
| Max speed to fill a car in one pass | 4.306 m/s to the brim, **4.784 m/s to the 90 % gate** |
| Accretion drag at 1 m/s | 27.8 kN |
| Loading terminal speed `sqrt(P/ṁ)` | 6.000 m/s — **above** the one-pass limit |
| Accretion drag at that speed | 166.7 kN (adhesion cap 441.3 kN) |
| Full throttle, 30 m approach | 10.5 s, reaching 4.95 m/s |

### 3.1 The timescale choice — stated explicitly

**There is no time compression. `C = 1`: one second of wall clock is one second for the
train.** The clock is honest and nothing in the simulation is sped up or slowed down.

Earlier versions of this document specified `C = 25`, because 2000 t at a realistic
2500 t/h takes 48 minutes and that is not a game. The brief's §3.8 offered three ways out —
compress time, reduce car capacity, or raise the flow rate — and this build initially took
the first. It now takes the third instead (§3.2): at 100 000 t/h the whole load is 72 s, so
the compression had nothing left to do.

Removing it was not merely tidy, it was necessary. **Every timing window in the game scales
as `1/C`**, and with the flow rate raised 40× the windows had become unplayable:

| window | at `C = 25` | at `C = 1` |
|---|---|---|
| fill one car to the 90 % gate | 130 ms | **3.24 s** |
| coupling gap passes, at 0.5 m/s | 120 ms | 3.00 s |
| coupling gap passes, at 1.0 m/s | 60 ms | 1.50 s |
| car under the chute, at 1.0 m/s | 620 ms | 15.5 s |

A 137 ms fill window is shorter than human reaction time, so the chute could not be opened
and shut for a single car at all. The design requirement, set by Shayne, is that filling a
car take **at least 3 s of wall clock**; `C = 1` gives 3.42 s.

Consequences, all accepted:

- The physics step stays `h = 1/240 s` of simulated time, which is now also 1/240 s of wall
  clock: **4 steps per rendered frame at 60 fps**, down from 100. Accuracy is unchanged and
  the cost is negligible.
- Simulated speeds are real speeds and now *look* like real speeds. A 1 m/s crawl reads as
  a 1 m/s crawl.
- **The train feels 25× more sluggish than it did**, because perceived responsiveness goes
  as `C × v` and `C` fell by 25. Full throttle from rest now covers 0.18 m in the first
  second and takes 29 s to cover the 30 m approach. The 100× power cut of §3.2 was made to
  tame twitchiness at `C = 25` and is, at `C = 1`, an overcorrection. See open point 6.

### 3.2 Departures from the brief's own text

Two numbers here contradict the brief directly, rather than filling in something it left
open. Both were made on play feedback from Shayne and both are disclosed here and in the
player-facing intro.

**Rated power 30 kW, against the brief's 2–4.5 MW band — 100× lower.** With a 30 m approach
there is no line haul to do: the entire game is low-speed shunting, placing a 15.5 m trough
under a 0.8 m chute, twenty times. Under 25× time compression a 3 MW unit crossed a car
pitch faster than anyone could react. Cut 10×, played, cut 10× again. The effect is to move
the adhesion/power crossover from 6.80 m/s down to 1.13 m/s, so the train is power-limited
over its entire working range and adhesion governs only the first tenth of a second off the
mark. Nothing else about the locomotive changed: it is still 150 t on 150 t of driven axles
at μ = 0.30.

**Approach 30 m, against the brief's 1 km.** At the speeds above, 1 km was a minute of
holding full power with nothing to judge and nothing to learn. The approach was never where
the physics lived; the chute is. Total travel over a full run is still 383.3 m, because the
train is 361 m long and every car has to reach the chute.

**Chute flow 100 000 t/h, against the brief's 1500–3000 t/h — 40× higher.** This one is
forced by the brief's own physics. At a realistic 2500 t/h a 100 t car takes 144 s to fill,
while its trough is only 15.5 m long, so a car can be filled in a single pass only below
0.108 m/s. At that speed the `v·dM/dt` term the entire brief is built around is **74.7 N**
against a 710 t train — a deceleration of 1.05×10⁻⁴ m/s², invisible. The realistic flow rate
makes the headline effect unobservable, and makes stop-and-fill the only viable strategy,
in which case `v = 0` and there is no accretion drag at all.

At 100 000 t/h a car fills in 3.6 s and can be filled in one pass at up to 4.31 m/s. The
accretion drag at 1 m/s is then **27.8 kN**, and it rises linearly with speed, so full
throttle with the chute open has an emergent terminal speed

```
v_eq = sqrt( P_rated / ṁ_flow ) = sqrt( 1 MW / 27 777.8 kg/s ) = 6.000 m/s
```

which is a genuinely pleasing consequence of the model rather than anything that was put
in. **At 1 MW that terminal speed sits above the 4.31 m/s one-pass limit**, so the grain no
longer holds the train inside the fillable band on its own; the player must. See §9 item 6.

This is the brief's §3.8 trade-off taken in the direction it left open: it names raising
the flow rate as one of the three permitted ways to fix the timescale.

**No friction brake** (§2.5) is a third departure, though the brief only constrained how a
brake must be modelled rather than requiring one. The regenerative model still satisfies
both of its stated constraints: a force, and adhesion-limited.

---

## 4. Integration

### 4.1 Scheme

- **Fixed timestep with an accumulator, fully decoupled from the render loop.** A raw
  `requestAnimationFrame` delta is never fed to the physics.
- **Base step `h = 1/240 s of simulated time.`** Chosen so accuracy is independent of the
  compression factor and of display refresh rate. At C = 25 this is 6000 steps per wall
  second, 100 steps per frame at 60 fps. Each step is one rigid body plus an `O(1)` chute
  overlap computation — a few dozen floating point operations — so the cost is well under a
  millisecond per frame. Justification for 1/240 rather than 1/60: the ledgers must close to
  0.1 %, and the traction force varies as `P/v`, which is stiffest exactly where play
  happens, at low `v`.
- **Adaptive substepping inside each base step.** The stepper *marches* across the base
  step, sizing each substep directly from the instantaneous force so the predicted velocity
  change stays inside `|Δv| ≤ 0.01·max(|v|, v_ref)`, with `v_ref = 0.05 m/s` as a floor so
  the criterion stays finite at `v → 0`:

  ```
  dt = sub_tol · max(|v|, v_ref) · M / |F_ext| ,   clamped to the base step remainder
  ```

  This is what keeps `F = P/v` tractable near `v = 0`, and it is part of the core integrator
  used in normal play — not a path added for the tests.

  **Corrected at task 4.** This document originally specified recursive *bisection* to a
  depth cap of 20. That cannot do the job, and the figures are worth recording. Test 3
  starts at `v_0 = 10⁻⁶ m/s` with `μ → ∞`, so `F = P/v = 3×10¹² N` and the substep must be
  `1.18×10⁻¹⁰ s` against a base step of `4.17×10⁻³ s` — 26 levels of bisection, or
  6.7×10⁷ substeps. A depth cap of 20 would *both* truncate the launch and cost 10⁶
  substeps. The marching stepper resolves the same launch in **237 substeps**, and averages
  1.029 substeps per base step over a 60 s run. `sub_depth` is therefore replaced by
  `sub_guard`, a loop guard of 200 000 iterations that should never be approached; the test
  panel reports the maximum substeps per base step so a silent truncation cannot pass
  unnoticed.
- **Integrator**: classical RK4 on the state `(x, p)` with `M(t)` advanced from captured
  grain. Momentum form, per §2.3.
- **Accumulator clamp**: at most 3 frames' worth of simulated time is caught up in one
  render, so a stalled tab cannot trigger a catch-up spiral.
- **Ledger terms accumulate inside the physics step only.** Never in the render step.

### 4.2 Determinism

The integrator is a pure function of `(state, control values, h)`. Control values from the
continuous slider are **sampled once per rendered frame and held constant across all
substeps of that frame**, so a given control-value time series always produces an identical
trajectory. No `Math.random` anywhere in the physics; the cosmetic grain particles use a
seeded PRNG kept in a separate stream that the physics never reads.

The test harness sets `throttle`, `brake` and `chute` programmatically, bypassing the input
layer entirely, so every test in §7 is exactly reproducible.

---

## 5. What the player sees

### 5.1 Layout

Single `<canvas>` for the world, scaled by `devicePixelRatio`; DOM overlay for HUD, fill
strip and popups, so text stays crisp and accessible.

**Play view.** Locked so the chute sits at ~55 % of viewport width. Scale ~12 px/m on a
400 px phone, showing about 33 m of track — two car bodies and the gap between them, at a
scale where the grain stream, the coupling gap and the fill level are all plainly readable.
Ground line at ~80 % height.

**Fill strip.** Persistent, all 20 cars at a glance. Top edge in portrait, right side in
landscape and on desktop. Per car: a vertical fill bar, the **pass threshold as a marked
line** (drawn from `fill_target`, never a literal), car number, and a highlight on whichever
car is currently under the chute. Coupling
gaps are drawn as gaps, so a chute sitting over a gap is visible in the strip as well as in
the play view.

**Clear indication of what is under the chute.** Three simultaneous cues: the highlight in
the fill strip; a caption reading `CAR 8` or `GAP 8–9` or `LOCO` or `NO CAR`; and the
grain stream itself drawn either landing in a trough or spilling to the ground.

### 5.2 HUD (always visible)

- Speed, m/s.
- Elapsed wall time as `m:ss.s`, with simulated time beside it.
- Total instantaneous mass `M`, in tonnes.
- Grain loss as a percentage of grain released, with the 1 % budget as a bar.
- **Accretion retarding force `v·dM/dt`, in kN, as a live bar.** Always on screen, because
  it is the physical heart of the game and the player should watch it grow when loading on
  the move.
- One tap expands the full ledger panel: `W_traction`, `W_regen`, `ΔKE_total`, `E_diss`, all
  three mass buckets, and the live closure percentages of Tests 1 and 2.

### 5.3 Controls

**Continuous slider** (decision 9), full brake at one end through neutral to full power at
the other, plus a reverse toggle and a chute toggle.

- Slider is **sticky**: it holds where released, like a locomotive controller. It does not
  spring back to neutral.
- Positioned on the right edge in portrait with a left-handed mirror option, so the thumb
  does not cover the chute.
- Mapping: neutral in the centre; toward one end `throttle ∈ (0, 1]` with `brake = 0`;
  toward the other `brake ∈ (0, 1]` with `throttle = 0`. Throttle and brake are never both
  non-zero.
- Reverse toggle sets the traction sign. Only actuable below 0.2 m/s.
- Chute toggle: a large button, plus SPACE.
- Keyboard: arrow up/down nudge the slider by 0.05, `0` neutral, `Home`/`End` full power and
  full brake, `R` reverse, `SPACE` chute, `P` pause, `D` done.
- Pause freezes everything including both clocks; controls are inert while paused.

### 5.4 Run structure

Clock starts on the first non-neutral control input or first chute opening. Free shunting
throughout. Run ends when the player presses **DONE** (confirmed), and the clock runs until
then, so an undersized car forces a real decision: reverse and pay the time, or accept the
fail.

**End-of-run report.** Both gates with their actual figures and the offending cars named;
elapsed time; grade band; the full ledger with Test 1 and Test 2 closures; and personal best
comparison.

### 5.5 Modes

- **Timed run.** The scored mode.
- **Practice.** No clock, no gate, free reset, jump-the-train-to-any-car control, ledger
  panel expanded by default.
- **How it works.** The intro popups, re-openable at any time.
- **Debug.** The self-test panel of §7.

`localStorage` holds only the best passing time and the "intro seen" flag, both resettable
from the title screen. Reads and writes are wrapped, and the game functions normally if
`localStorage` throws or is empty.

### 5.6 Scoring

```
Gate 1:  min over cars of (fill / capacity)  ≥  90.0 %   (fill_target, §9 item 7)
Gate 2:  m_ground / m_released              <   1.00 %
```

Both pass → **PASS**, ranked by elapsed time alone. Either fails → **FAIL**, with the
specific cars and the actual loss figure reported. No composite score, no partial credit:
the brief's two thresholds stay thresholds.

**Grade bands, calibrated 2026-09-12** (PLAN task 22, now done):

| grade | elapsed |
|---|---|
| A | < 1:45 (105 s) |
| B | < 2:05 (125 s) |
| C | < 2:30 (150 s) |
| D | otherwise |

Set against a **measured floor**, not invented. An ideal driver — exact speed hold, perfect
chute timing — completes a passing run in **90.8 s**, holding 4.50 m/s, every car at 90.7 %
with 0.000 % loss. At 4.60 m/s the run takes 89.0 s but every car falls under the gate, so
~91 s is a hard physics floor: below it no chute timing can fill the cars, because a car is
not under the chute long enough. A is therefore within about 15 % of the floor.

The previous bands (A < 2:30, B < 3:00, C < 3:45) were invented before anyone had played a
complete run, and were far too loose — they awarded an A to a 131 s run the player described
as "OK". Full sweep in `docs/MEASUREMENTS.md` §1.

### 5.7 Art

Warm storybook cartoon, all procedural Canvas 2D, no image assets.

| Role | Colour |
|---|---|
| Outline | `#3B2A1F`, 2–3 px, on every silhouette |
| Grain, fills | `#E8B54A` |
| Locomotive body | `#C4483A` |
| Sky | `#7FC4CE` → `#BFE3E8` gradient |
| Chute tower, silo | `#A89684` |
| Soil, spill piles | `#8A6F52` |

Rounded chunky forms, one soft gradient per fill, thick outlines throughout — outlines are
what keep the coupling gaps and the threshold line readable when a car is only a few dozen
pixels wide.

Spilled grain accumulates in a heightfield of 2 m bins along the track and is drawn as
mounds, so the player can see where they wasted it.

Performance target: 60 fps on a mid-range phone with all 20 cars and grain in flight.
Cosmetic grain particles are drawn from a fixed-size pool, capped at 400.

### 5.8 Intro popups

A short skippable sequence at first start, re-openable from the title screen and from pause.
Four cards. **The physics wording was a draft for Shayne to replace; he accepted it as
written on 2026-09-11.** The draft marking, its red bar and the `.draft` / `.draftlab`
styling are gone, and `INTRO` entries no longer carry a `draft` flag.

The wording states, and is still checked against, these constraints:

- Zero rolling resistance and zero drag is an idealisation, **and the train therefore never
  slows on its own — every stop must be braked.**
- Time is **not** compressed (`C = 1`, §3.1); all other quantities are real except the two
  disclosed departures below.
- Grain arrives with no forward speed, so loading pushes back on the train, harder the
  faster you go. **Not called the rocket equation.**
- Momentum is conserved **only while coasting** — the caveat is never dropped.
- The rules: ≥ 90 % every car, < 1 % loss, as fast as possible.
- The controls.
- The two disclosed parameter departures — flow rate and rated power (§3.2) — and, since
  2026-09-11, the 90 % pass threshold (§9 item 7).

One number in the accepted text was corrected on acceptance rather than left as written: it
read "a loaded train needs about 30 m to stop from 1 m/s", which was a 30 kW-era figure. At
1 MW the measured distances are 3.1 m from 1 m/s and 77 m from 4.3 m/s loaded, so the text
now quotes the loading-speed figure. The claim it supports — "creeping is not optional" — is
if anything stronger: 77 m is four and a half car pitches.

---

## 6. Code structure

One file, in this order: `<style>`, markup, `<script>`. Within the script, as separate
closures with no shared mutable state beyond an explicit `world` object:

| Module | Responsibility |
|---|---|
| `PARAMS` | Every constant from §3, frozen. Single source of truth. |
| `physics` | State, RK4 momentum-form integrator, adhesion and brake limits, adaptive substepping. Knows nothing about rendering. |
| `geometry` | Car positions from train position, chute overlap, capture and spill per step. |
| `ledger` | The accumulators of §2.8 and the closure computations. **Written from §2.7 route 2, independently of `physics`.** |
| `render` | Canvas 2D drawing. Reads state, never writes it. |
| `input` | Slider, buttons, keyboard. Produces control values; sampled once per frame. |
| `ui` | HUD, fill strip, popups, reports. |
| `tests` | §7. Drives `physics` directly. |
| `LTT` | The console API of §7.5. |

`physics` and `ledger` must not import each other's algebra. That separation is the brief's
explicit safeguard against the factor-of-two error and is a review criterion, not a style
preference.

---

## 7. Tests

Acceptance criteria. Every closed form below was derived in this document, independently of
any code. Values quoted are the predictions to check against.

**No test may be weakened to pass, no expected value hard-coded from an observed output, and
no exception caught to make a failure disappear. A failing test is reported as a failure and
the build stops.**

### 7.1 Test 1 — mass ledger

```
m_released  =  m_cars + m_ground
```

to within 0.1 % over a full run. All three exposed live in the ledger panel and in the
end-of-run report.

Because landing is instantaneous (decision 5) there is no in-flight bucket and the identity
holds at **every** physics step, not merely at the end — a strictly stronger statement than
the brief requires. The harness asserts it every step during test runs.

Run: a scripted 400 s simulated sequence that deliberately opens the chute over gaps, over
the loco, past the end of car 20, and over an already-full car, so every loss channel is
exercised.

### 7.2 Test 2 — energy ledger

```
W_traction − W_regen  =  ΔKE_total + ½ ∫ v²·(dM/dt) dt
```

`W_regen` plays the role of the brief's `W_brakes`: it is the energy the motors took back
out of the train. The split is exact rather than approximate — `max(a,0) + min(a,0) = a`
identically and Simpson's rule is linear, so `W_traction − W_regen` is exactly the Simpson
integral of `F·v`.

to within 0.1 %. `ΔKE_total` uses the full instantaneous mass including loaded grain. **Every
term reported separately**: `W_traction`, `W_regen`, `ΔKE_total`, `E_diss`, and the residual
both as an absolute value and as a fraction of `W_traction`.

Spilled grain enters neither side, which is a cross-check against Test 1: a run with large
spill must still close Test 2 to the same tolerance.

Run: the same scripted sequence as Test 1, with throttle and brake both exercised.

### 7.3 Test 3 — constant power, checkable by hand

Chute shut, brakes off, from rest, constant power `P`, constant total mass `M`:

```
v(t) = sqrt( 2·P·t / M )
s(t) = (2/3)·sqrt( 2·P/M )·t^(3/2)
```

These hold **only in the constant-power regime**. With the shipped `μ = 0.30` the adhesion
limit binds below `v_c = 6.798 m/s`, reached at `t_c = 10.94 s` from rest at tare mass, so
the formulas do not apply from `t = 0`.

**Per decision 13, Test 3 is run with `μ → ∞`.** The adhesion branch of the `min()` then
never selects, `F_traction = P/v` exactly, and the brief's closed forms hold verbatim from
`t = 0`. The `min()` code path itself is unchanged; only a parameter takes a limiting value.

The `v → 0` singularity is handled by the core integrator's adaptive substepping (§4.1). The
run starts from `v_0 = 1×10⁻⁶ m/s` rather than exactly zero, because `P/0` is infinite; the
exact solution with that initial condition is `v = sqrt(v_0² + 2Pt/M)`, and `v_0² = 10⁻¹²`
sits fifteen orders of magnitude below `v²` at `t = 1 s`, so it is indistinguishable from the
brief's formula at the 0.1 % tolerance. **This is stated, not hidden** — see open point 3.

Predictions at `M = 710 t`, `P = 1 MW`, `sqrt(2P/M) = 1.678363`:

| `t` (s) | `v` (m/s) | `s` (m) |
|---|---|---|
| 10 | 5.30745 | 35.3830 |
| 30 | 9.19277 | 183.8554 |
| 60 | 13.00054 | 520.0217 |

**Supplementary check S3 — the adhesion limit itself.** Beyond the brief's four tests, and
clearly labelled as an addition Shayne may strike. Because Test 3 runs at `μ → ∞`, nothing
in it exercises the adhesion cap that governs all normal play. S3 runs the shipped model
(`μ = 0.30`) from rest and compares against the exact piecewise solution derived here:

```
phase 1,  t ≤ t_c:   v = a·t ,                         a = μ·m_driven·g / M
phase 2,  t ≥ t_c:   v = sqrt( 2·P·(t − t_c/2) / M )
```

The offset is **exactly `t_c/2`**, because a constant force from rest delivers precisely half
the energy that constant power would over the same interval:
`½Mv_c² = ½·(F_a·v_c)·t_c = ½·P·t_c`. A pleasing result, and easy to check by hand.

At `M = 710 t`: `a = 0.621548 m/s²`, `t_c = 3.645793 s`, `v_c = 2.266036 m/s`,
`s_c = 4.1307 m`, offset `t_c/2 = 1.822896 s`.

| `t` (s) | exact `v` | exact `s` | naive `v` | naive `s` |
|---|---|---|---|---|
| 10 | 4.79938 | 27.5403 | 5.30745 (+10.6 %) | 35.3830 (+28.5 %) |
| 30 | 8.90910 | 168.7321 | 9.19277 (+3.2 %) | 183.8554 (+9.0 %) |
| 60 | 12.80153 | 497.8809 | 13.00054 (+1.6 %) | 520.0217 (+4.4 %) |

The naive deviation decays only as `t_c/2t`, still 0.30 % at `t = 600 s`, which is why
Test 3 needs `μ → ∞` rather than simply being evaluated late. The panel prints this
comparison so the discrepancy is on the record rather than buried.

### 7.4 Test 4 — coasting accretion, exact, no integration needed

Chute open, zero traction, zero brake. Then `Mv` is constant and

```
v_final          = v_0 · M_0 / M_final
KE_final / KE_0  = M_0 / M_final
```

Prediction, from `v_0 = 2 m/s`, `M_0 = 710 t`, chute open 60 s simulated onto car 1
(41.667 t captured, well inside the 100 t capacity, so no overflow):

| Quantity | Predicted |
|---|---|
| `M_final` | 751.6667 t |
| `p = Mv` | 1 420 000 kg m/s, constant |
| `v_final` | 1.889135 m/s |
| `KE_final/KE_0` | 0.94456763 |
| `ΔKE` | −78 713.97 J |
| `½∫v²(dM/dt)dt` | +78 713.97 J |

Note that this passes to near machine precision with the momentum-form integrator, since `p`
is advanced by a zero impulse. **That is the point, not a fudge**: the momentum form is the
honest statement of §2.3, and it makes a wrong accretion term a structural impossibility
rather than a tuning question. To ensure the test still has teeth, two independent
cross-checks are reported alongside:

- **Accretion impulse.** `∫ v·(dM/dt) dt = p·ln(M_final/M_0) = 80 979.75 N·s`. Derived from
  `∫M dv = −∫v dM` with `v = p/M`. This is accumulated separately by `ledger` as `J_acc`,
  not by `physics`, so agreement is a genuine two-route check.
- **Test 2 closure under coasting.** With `F_ext = 0`, Test 2 reduces to
  `0 = ΔKE + ½∫v²(dM/dt)dt`. The two figures above sum to zero analytically to
  8.7×10⁻¹¹ J, so Test 4 and Test 2 cross-validate each other, and a factor-of-two error
  would show up here as a factor-of-two residual.

### 7.5 How the tests are run

Both routes live in `app/index.html` (decision 7).

**Panel.** Reached by the URL `app/index.html?test`, or the DEBUG button on the title screen.
A button per test; each prints, on the page and to the console, a table of computed value,
closed-form prediction, absolute difference and relative error, with a PASS/FAIL per row
against the 0.1 % tolerance. Plus a RUN ALL button.

**Console API,** so Shayne can substitute his own `P`, `M` and `t` and compare against a
figure he worked out himself:

```js
LTT.setState({ throttle, brake, chute, reverser, v, x, carFill })
LTT.step(dt_sim)                  // advance exactly dt_sim of simulated time
LTT.run({ t, throttle, brake, chute, mu, M, P, v0 })
LTT.runTest(n, opts)              // n = 1..4, or 'S3'; opts override any PARAM
LTT.ledger()                      // every accumulator plus both closures
LTT.params()                      // the frozen PARAMS
LTT.reset()
```

`opts` accepts `mu: Infinity`, which is exactly how Test 3 is invoked.

---

## 8. Out of scope

Explicitly not built, so review can confirm the boundary:

- The project website at the repository root. **The root is untouched.**
- Coupling slack, buffer dynamics, slack action.
- Grade, curvature, wind, weather.
- Rolling resistance and aerodynamic drag (§2.1, disclosed).
- Multiple locomotives, distributed power, dynamic or regenerative braking.
- Sound.
- Any network request, analytics, or external asset.
- Any server-side leaderboard; the best time is local only.

---

## 9. Open points for review

Flagged rather than decided, for Shayne's call:

1. ~~**Grade bands** are provisional~~ — **closed 2026-09-12.** Calibrated against the
   measured ~91 s floor and approved by Shayne: A < 105 s, B < 125 s, C < 150 s (§5.6).
2. **Supplementary check S3** (§7.3) is beyond the brief's four tests. Kept because Test 3
   at `μ → ∞` leaves the adhesion cap unverified, but easy to strike.
3. **Test 3's `v_0 = 10⁻⁶ m/s`** is a departure from "from rest", forced by `P/0`. The exact
   solution with that initial condition is quoted and the difference is `O(v_0²)`, fifteen
   orders below the tolerance — but it is a departure, and is stated as one.
4. **Power is below the brief's band and the approach is 30 m, not 1 km** (§3.2). Both
   are deliberate playability calls made on play feedback, and both are departures from the
   brief's own text rather than from something it left open.
5. ~~**Physics wording in the intro is a draft**~~ — **closed 2026-09-11**, accepted as
   written (§5.8). One stale stopping distance was corrected on acceptance; see §5.8.
6. **Rated power raised to 1 MW** (30 kW → 500 kW → 1 MW), on Shayne's instruction. This
   **crosses the accretion-drag knee deliberately**: `v_eq = sqrt(P/ṁ) = 6.000 m/s` against
   a one-pass fill limit of 4.306 m/s, so full throttle now outruns the chute and the grain
   no longer governs on its own. Measured, one full-throttle pass from the start line with
   the chute held open, against the shipped code:

   | `P_rated` | pass time | settled `v` | grain aboard | worst car |
   |---|---|---|---|---|
   | 500 kW | 91.9 s | 4.46 m/s | **94.9 %** | 91 t |
   | 1 MW | 69.4 s | 6.24 m/s | **69.7 %** | 68 t |

   At 500 kW holding full throttle all the way very nearly met the 95 % target by itself; at
   1 MW it does not, and the player has to modulate the throttle to stay under 4.31 m/s over
   each car. The settled speed exceeds `v_eq` slightly because grain is captured only over
   the car bodies — 91.18 % duty — so the effective flow is `0.9118·ṁ` and
   `v_eq/sqrt(0.9118) = 6.28 m/s`, which is what is measured.
   **This invalidates the grade bands (open point 1) more than `C = 1` already did.**
7. **Pass threshold lowered 95 % → 90 %** (2026-09-11), on Shayne's instruction. **This is a
   departure from the brief**, which states the objective as "keep every car above 95 % full"
   and marks the 95 % threshold on the fill strip. It is disclosed to the player in the intro
   and in the end-of-run report, both of which now read 90 %.

   What it buys, measured against the shipped code by holding a constant speed past the whole
   train with the chute open (an ideal driver — the physics, not the input layer):

   | held speed | median car fill |
   |---|---|
   | 4.30 m/s | 100.0 % |
   | 4.50 m/s | 95.7 % |
   | **4.78 m/s** | **90.1 %** |
   | 5.00 m/s | 86.1 % |

   So the fastest single pass that still passes goes from **4.53 m/s to 4.78 m/s**, 5.6 %
   quicker. It does *not* make full throttle viable: at 1 MW that settles at 6.24 m/s and
   loads 69.7 %, well under either gate. The game still requires throttle modulation.

   The clean algebraic statement is `v_max = L_body / (fill_target · m_cap / ṁ)`, which the
   measurement above confirms to three figures.
