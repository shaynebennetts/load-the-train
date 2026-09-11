# Testing, the harness, and driving the app headlessly

Companion to `../HANDOVER.md`.

---

## 1. Running the tests

```
app/index.html?test                      panel, all tests
LTT.runTest(3, {mu: Infinity, P_rated: 4e5, M: 5e5, t: 30})     any override
LTT.ledger()  LTT.params()  LTT.derived()  LTT.geom
LTT.state()   LTT.setState({...})  LTT.step(dt)  LTT.reset()  LTT.arm()
```

Or click **DEBUG** on the title screen.

## 2. The headless harness

Runs the **actual shipped code**, not a copy. The scratch scripts are not in the repository;
this is all of it:

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
```

```js
// runall.js
const {loadCore} = require('./harness.js');
for (const t of loadCore().runAll())
  console.log((t.pass ? 'PASS' : 'FAIL') + '  ' + t.id + '  ' + t.title);
```

`runAll()` returns an **array** of `{id, title, rows, pass}`. There is no `.text` field — an
earlier version of this note said there was. Each row is
`{label, got, pred, unit, relerr, pass}`.

### Driving the physics from the harness

`setState({fill, x, v, throttle, chute})`. Two things to know:

- **`fill` is in kilograms per car, not a fraction.** `fill: [1,1,…]` gives a train 20 kg
  heavier than tare, not a full one. Use `m_cap` (100 000).
- **`setState` applies `fill` before `v`**, so `v` is converted to momentum at the right mass.
- `LTT.arm()` resets the ledger. Call it after teleporting the train, or the loss meter shows
  the grain you spilled while setting up.

## 3. Rules, from the brief, which must not be relaxed

- Do not weaken a test to make it pass.
- Do not hard-code an expected value from an observed output.
- Do not catch an exception to make a failure disappear.
- If a test fails, report the failure and stop.

**One trap already found and fixed — do not reintroduce it.** `row()` used to fall back to the
raw absolute difference when the expected value was zero, so `ΔKE + E_diss = 0` compared
**joules against a dimensionless 0.1 % tolerance**. It passed only by luck of magnitude and
broke the moment the flow rate rose 40×. Zero-expectation rows now **require** a `scale`, and
`row()` throws without one. Keep that.

---

## 4. Screenshots and animation, headless

Used for the project website and the wall GIF. Copy `app/index.html` to a temp file and inject
before `#ltt-core`:

```html
<script>try{localStorage.setItem("ltt.seen","true");}catch(e){}</script>
<style>#app{width:1280px !important;height:800px !important;}</style>
```

then drive the real UI after load — `btn('Timed run').click()`, `LTT.setState({...})`,
`document.getElementById('bchute').click()`, `#bledg` for the ledger drawer — and shoot with
`--headless=new --screenshot`.

Pin `#app` explicitly: headless lays out ~80 px wider than it screenshots, which silently
clips the right-hand controls.

### Four traps, all of which cost time

1. **rAF does not fire under `--virtual-time-budget`.** The game survives this because
   `schedule()` also has a 40 ms timer (see `PHYSICS.md` §6). An app that relies on rAF alone
   renders one frame and freezes — that is exactly what happened with `pendulum-example`, and
   the fix there was to drive the physics synchronously instead.
2. **Give every Chrome launch its own `--user-data-dir`.** Rapid sequential launches sharing
   the default profile produce intermittently blank or incomplete renders.
3. **A late `resize` event clears the canvas.** Any app whose resize handler reassigns
   `canvas.width` wipes the drawing buffer; if nothing repaints afterwards the screenshot is
   blank. Add a redraw on `resize` in the injected script.
4. **Frame sequences for a GIF**: because the sim is deterministic, render frame *k* by
   advancing exactly *k* steps in a fresh page, passing *k* in the query string. Then
   `ffmpeg -framerate 15 -i fr%03d.png … -loop 0 out.gif`. Check for blank frames by counting
   dark pixels before assembling — a silently blank frame is otherwise invisible until the
   GIF is on a projector.
