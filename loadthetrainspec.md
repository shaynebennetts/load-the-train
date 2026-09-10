# Load the train — build brief

**Read this whole file before you do anything. This is a brief, not a specification.**
Your first job is to ask me questions until you can write the specification yourself.
Do not write any code until I have reviewed and approved `SPEC.md` and `PLAN.md`.

Author: Shayne Bennetts. Course: Basic Skills for Experimentalists (TIGP 2026), Project 1.

---

## 1. Vision

"Load the train" — a game for loading a freight train.

A train with 20 carriages and a locomotive starts 1 km from the loading chute, which can
be turned on and off. The player controls the power of the locomotive, its throttle, and
can turn the chute on and off to turn on the supply of grain. Make reasonable assumptions
about the mass of the locomotive, its maximum power, its braking power, and the flow rate
of the chute dropping grain.

The aim of the game is to load the train as fast as possible, to achieve more than 95 %
loading of each car, and less than 1 % grain loss.

We need to do the physics of it. The game is web based so it can be deployed on a GitHub
Pages site, all in a single HTML file. The graphics should be compelling, cartoon style
and visually appealing, with feedback on losses and car filling level. It must be playable
on a phone or in a browser. The physics, rules and controls are explained in a brief
introduction as popups when the user starts.

---

## 2. Deliverables and layout

| Path | What it is |
|---|---|
| `app/index.html` | The game. A single self-contained file. |
| `SPEC.md` | Written by you, reviewed by me, before any code. |
| `PLAN.md` | Written by you. Small tasks, each with a testable deliverable. |
| `index.html` (root) | The project website. **Not yours to write.** Leave the root alone. |

Constraints:

- Plain JavaScript, HTML and CSS. **No build step. No external dependencies. No CDN
  loads.** The file must work opened directly from disk and served from GitHub Pages.
- Everything in one file: markup, styles, script, and any embedded art.
- Do not put the game at the repository root. The root is reserved for the project page.
- Target 60 fps on a mid-range phone with all 20 cars and grain in flight.

---

## 3. Physics

### 3.1 Assumptions

Zero rolling resistance and zero aerodynamic drag. This is a deliberate idealisation and
it must be disclosed to the player, along with its consequence: the train never slows down
on its own, so every stop has to be braked.

### 3.2 This is mass accretion, not the rocket equation

Do not describe this as the rocket equation anywhere, including in player-facing text.

The general variable-mass law is:

```
F_ext = d(Mv)/dt - u*(dM/dt)
```

where `u` is the velocity of the mass being added or removed.

- A rocket **ejects** mass with a relative exhaust velocity. That term produces thrust.
- This train **accretes** grain arriving with zero horizontal velocity, so `u = 0`. The
  term vanishes and what remains **retards** the train. Opposite sign to a rocket.

The textbook analogue is sand falling onto a moving conveyor belt, or a hopper car loaded
in motion.

### 3.3 Equation of motion

With `u = 0`:

```
F_traction - F_brake = d(Mv)/dt = M*(dv/dt) + v*(dM/dt)
```

The `v*(dM/dt)` term is a real retarding force, proportional to loading rate and speed.
**It must emerge from the integrator. It must never be a tuned penalty constant.** It is
the physical reason that loading while moving costs you, which is the core tension of the
game.

`M(t)` is the instantaneous total mass: locomotive plus car tare plus grain loaded so far.

### 3.4 Traction and the adhesion limit

Traction force at constant power is `P/v`, which diverges as `v` approaches zero. The model
needs an adhesion limit:

```
F_traction = min( P_available / v , mu * m_driven * g )
```

where `m_driven` is the weight on the driven axles. State your assumed `mu` in `SPEC.md`.
Without this the train launches off the line at `t = 0`, or you get a NaN.

Brakes are also adhesion-limited. Model the brake as a **force**, or as a fraction of
maximum brake effort, not as a constant power, and report its energy as the integral of
`F_brake * v` over time. A constant-power brake behaves unphysically at low speed for the
same reason traction does.

### 3.5 Momentum

Momentum of train plus grain is conserved **only while coasting**, when traction and brake
force are both zero. Under throttle or brake the rails supply an external horizontal force
and `d(Mv)/dt = F_ext`. Do not claim general momentum conservation in player-facing text.

### 3.6 Energy, and the factor of two

Grain arriving at rest onto a car moving at speed `v` absorbs work from the train at rate
`v^2*(dM/dt)`. **Exactly half** of that becomes kinetic energy of the grain; the other half
is dissipated in the inelastic momentum transfer.

| Term | Rate |
|---|---|
| Work absorbed from the train | `v^2*(dM/dt)` |
| Kinetic energy gained by the grain | `0.5*v^2*(dM/dt)` |
| Dissipated as heat | `0.5*v^2*(dM/dt)` |

Loaded grain then travels with the train, so its kinetic energy is already inside the total
kinetic energy. Only the dissipated half is a separate ledger line.

Getting this factor of two wrong is the single most likely physics error in this build.
Derive it independently before you code it, and do not let the same reasoning that writes
the integrator also write the ledger.

Grain that misses a car is never accelerated and contributes to neither energy term.

### 3.7 Grain loss must be geometric

Loss falls out of chute position, car geometry and train speed. Grain leaving the chute
lands where the chute is; if a coupling gap, or the space beyond the last car, is under the
chute, that grain is lost. **No tuned loss coefficient.** A fitted penalty would still feel
like a game and would fail every measurement made of it.

Model at minimum: chute horizontal position and aperture width, car length, gap length
between cars, per-car fill level and capacity, and overflow once a car is full.

### 3.8 Parameters

You choose these and **state every value and its justification in `SPEC.md`**. Real figures
for magnitude checking, not values you must adopt:

| Quantity | Realistic range |
|---|---|
| Freight locomotive mass | 120 to 200 t |
| Locomotive rated power | 2 to 4.5 MW |
| Wheel-rail adhesion coefficient | 0.25 to 0.35 dry rail with sanding |
| Grain hopper car tare mass | 25 to 30 t |
| Grain hopper car capacity | 60 to 100 t |
| Grain terminal loading rate | 1500 to 3000 t per hour |

Note the timescale problem before you pick: 20 cars at 100 t each is 2000 t, which at a
realistic 2500 t per hour takes about 48 minutes. That is not a game. Either compress time
by an explicit stated factor, or reduce car capacity, or raise the flow rate. **Say which
choice you made and state the factor.** Do not silently use unphysical numbers.

### 3.9 Integration

The ledger tests below have to close to 0.1 %, so integration accuracy is a requirement,
not a detail.

- **Fixed timestep** with an accumulator, decoupled from the render loop. A raw
  `requestAnimationFrame` delta makes the ledgers noisy and the tests irreproducible.
- A physics step of the order of 1/240 s, with substepping. Justify what you pick.
- Deterministic: the same inputs must give the same trajectory, so tests are repeatable.
- Accumulate ledger terms inside the physics step, never in the render step.

---

## 4. Tests

These are the acceptance criteria. The specification must include them and the code must be
verifiable against them.

**Test 1 — mass ledger.** Grain released by the chute, flow rate times open time, equals
grain in cars plus grain lost on the ground, closing to within 0.1 % over a full run.
Expose all three quantities.

**Test 2 — energy ledger.** With zero resistance and zero drag:

```
W_traction - W_brakes = dKE_total + 0.5 * integral( v^2 * dM/dt ) dt
```

closing to within 0.1 %. `dKE_total` uses the full instantaneous mass including loaded
grain. **Report every term separately.** Spilled grain enters neither side, which is a
cross-check against Test 1.

**Test 3 — constant-power limiting case, checkable by hand.** Chute shut, brakes off, from
rest, at constant power `P` with total mass `M`:

```
v(t) = sqrt( 2*P*t/M )
s(t) = (2/3) * sqrt( 2*P/M ) * t^(3/2)
```

Valid only in the constant-power regime, so it must be evaluated after the adhesion limit
stops binding. Say in `SPEC.md` where that crossover is.

**Test 4 — coasting accretion, exact and needing no integration.** Chute open, zero
traction, zero brake. Then `M*v` is constant, so:

```
v_final = v_0 * M_0 / M_final
KE_final / KE_0 = M_0 / M_final
```

If the code disagrees with this, the accretion term is wrong.

**Provide a way to run Tests 3 and 4 so I can compare against my own hand calculation.** A
debug or test mode that sets traction, brake and chute state programmatically, runs for a
specified time, and prints the ledger terms alongside the predicted values.

Do not weaken a test to make it pass, do not hard-code an expected value, and do not catch
an exception to make a failure disappear. If a test fails, report the failure and stop.

---

## 5. Game design

**Objective.** Load fast, keep every car above 95 % full, keep total grain loss under 1 %.
These pull against each other, and that tension is the game: moving fast means the chute
runs while a coupling gap passes under it.

**Controls, working on both touch and mouse or keyboard.**

- Locomotive throttle, forward and reverse.
- Brake.
- Chute open and closed.

**Feedback the player needs.**

- Per-car fill level, all 20 cars visible at a glance, with the 95 % threshold marked.
- Running total grain loss, against the 1 % budget.
- Speed, and elapsed time.
- Clear indication of which car, or which gap, is currently under the chute.

**Presentation.** Cartoon style, visually appealing, readable on a phone screen. The train
is longer than the viewport, so solve the camera and the overview together: the player must
be able to see the chute closely and still judge the whole train's fill state.

**Intro popups.** A short sequence at first start explaining the physics, the rules and the
controls. It must state that zero rolling resistance and zero drag is an idealisation, and
that consequently the train never slows on its own. Keep them skippable and re-openable.

**I will write the final wording of the physics explanation myself.** Draft it, but mark it
clearly as a draft for me to replace. Two errors to avoid in any draft: calling this the
rocket equation, and claiming momentum conservation without the coasting caveat.

---

## 6. How to proceed

1. **Ask me questions** until you can write the specification. Ask about anything
   underspecified here, especially the parameter and timescale choices in 3.8, the camera
   and overview problem in section 5, and the scoring formula.
2. Write **`SPEC.md`**: what it does, what the player sees, the physics with the equations,
   every assumed parameter value with its justification, the integration scheme, and how
   each of the four tests is run and verified.
3. Write **`PLAN.md`**: the build broken into many small tasks, each stating what "done"
   looks like and which test or observation demonstrates it.
4. **Stop.** I read both. Then a separate Claude session that did not write them reviews
   them independently. Expect at least one iteration.
5. Only then, code, one task at a time, committing as you go, changing nothing outside the
   current task.

Do not write code before step 5. Do not write to the repository root. If you think
something in this brief is wrong, say so rather than working around it.
