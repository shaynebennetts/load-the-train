# Measured numbers, and how they were measured

Companion to `../HANDOVER.md`. Everything here came out of the **shipped** `app/index.html`
via the headless harness in `TESTING.md`, not from algebra and not from memory. Re-measure
rather than trust these if a parameter changes.

Dates are when the measurement was taken. Parameters at the time: `P_rated` 1 MW,
`fill_target` 0.90, `mdot` 27 777.8 kg/s, `C` 1, unless a row says otherwise.

---

## 1. How long a complete run takes — 2026-09-12

Nobody had ever played one end to end, so this was simulated with an **ideal driver**: an
exact speed hold plus perfect chute timing (open only while the aperture lies wholly over a
trough that still has room). It is the physics ceiling, not a human one.

| held speed | run time | min car fill | loss | verdict |
|---|---|---|---|---|
| 3.00 m/s | 132.7 s | 100.0 % | 0.000 % | pass |
| 4.00 m/s | 101.1 s | 100.0 % | 0.000 % | pass |
| 4.30 m/s | 94.6 s | 94.9 % | 0.000 % | pass |
| **4.50 m/s** | **90.8 s** | **90.7 %** | **0.000 %** | **pass — the floor** |
| 4.60 m/s | 89.0 s | 88.7 % | 0.000 % | fail, every car under 90 % |
| 4.78 m/s | 86.0 s | 85.4 % | 0.001 % | fail |

**~91 s is a hard floor.** Below it no chute timing can fill the cars, because the car simply
is not under the chute for long enough.

**Shayne's first complete run: 2:11 (131 s)** — 1.44× optimal, and a good result. Under the
old bands it scored an **A**, because they started at `A < 150 s`, while he described the run
as merely "OK". That mismatch was the evidence that the bands were too loose, and is what
prompted the calibration below.

Run length is therefore **90–135 s**. The old worry that a full run might be tediously long
was unfounded.

### The bands these produced — applied 2026-09-12

| grade | elapsed | |
|---|---|---|
| A | < 105 s | within ~15 % of the 91 s floor |
| B | < 125 s | |
| C | < 150 s | **a 131 s run lands here** |
| D | ≥ 150 s | |

Approved by Shayne and applied to `rules.BANDS` and `SPEC.md` §5.6 together. Note the
boundary: 131 s is a **C**, not a B — an earlier note in this project claimed otherwise and
was wrong.

---

## 2. Fill against held speed — 2026-09-12

Whole train past the chute at a held constant speed, chute open, median car:

| held speed | median car fill |
|---|---|
| 4.30 m/s | 100.0 % |
| 4.50 m/s | 95.7 % |
| **4.78 m/s** | **90.1 %** |
| 5.00 m/s | 86.1 % |

Confirms `v_max = L_body/(fill_target·m_cap/mdot)` to three figures. Lowering the gate from
95 % to 90 % moved the fastest passing single pass from **4.53 → 4.78 m/s**, 5.6 % quicker.

### One full-throttle pass, 500 kW vs 1 MW

From the start line with the chute held open, measured against the shipped code:

| `P_rated` | pass time | settled `v` | grain aboard | worst car |
|---|---|---|---|---|
| 500 kW | 91.9 s | 4.46 m/s | **94.9 %** | 91 t |
| 1 MW | 69.4 s | 6.24 m/s | **69.7 %** | 68 t |

At 500 kW, holding full throttle all the way very nearly met the target by itself. At 1 MW it
does not — which is the point of the raise. Lowering the gate to 90 % did **not** bring full
throttle back into range.

---

## 3. Stopping distance — 2026-09-12

Shipped integrator at full reverse effort until `v` reaches zero, chute shut:

| from | empty, 710 t | loaded, 2710 t |
|---|---|---|
| 0.5 m/s | 0.20 m | 0.77 m |
| 1.0 m/s | 0.80 m | 3.07 m |
| 2.0 m/s | 3.22 m | 12.28 m |
| 4.0 m/s | 16.52 m | 63.07 m |
| 4.3 m/s | 20.2 m | 77.1 m |
| 6.0 m/s | 52.50 m | 200.38 m |

The piecewise formula in `PHYSICS.md` §5 reproduces every row to 0.01 m. The old
constant-power-only form gave 0.24 / 0.90 / 57.81 / 195.12 m for the 1.0-tare, 1.0-loaded,
4.0-loaded and 6.0-loaded rows — an under-estimate throughout.

**At loading speed a loaded train needs about 77 m to stop — four and a half car pitches.**
That is the number the intro now quotes; it used to quote "30 m from 1 m/s", a 30 kW-era
figure that was wrong by 10×.

---

## 4. Launch, approach and crossover — 2026-09-12

| quantity | value |
|---|---|
| `v_cross = P/(μ·m_driven·g)` | 2.266 m/s (was 1.133 at 500 kW) |
| launch acceleration, tare | 0.6215 m/s² — adhesion-limited, unchanged by power |
| launch acceleration, loaded | 0.1628 m/s² — likewise |
| tractive effort at 1 m/s, full | 441.3 kN (adhesion cap; `P/v` would be 1000 kN) |
| full throttle, 30 m approach | 10.5 s, arriving at 4.95 m/s (was 12.5 s at 4.04) |
| first second from rest, full throttle | 0.31 m — unchanged, the first second is adhesion-limited |

---

## 5. Test agreements — worst relative error per test

Tolerance is 10⁻³ throughout.

| test | what it closes | worst relerr |
|---|---|---|
| 1 | mass ledger: released = in cars + on ground | 2.4×10⁻¹⁴ |
| 2 | energy ledger: `W_tr − W_br = ΔKE + ½∫v²dM` | 8.3×10⁻¹⁰ |
| 3 | constant power from rest, μ → ∞ | 3.5×10⁻¹⁰ |
| 4 | coasting accretion, `Mv` constant | 5.9×10⁻⁹ |
| S3 | adhesion limit, exact piecewise solution | 1.6×10⁻¹⁰ |
| D | determinism | exact |

These are the figures quoted on the project website. If they drift, the website is wrong too.
