# Manim.js AI Agent Guide (Repository-Specific)

> Scope: this guide is based on the current implementation in this repository (`Dimkarpenko/Manim.js`), not on Python Manim.

## 1) What Manim.js is and how it relates to p5.js

Manim.js is a **p5.js-based animation toolkit** for math/CS visuals inspired by 3Blue1Brown style scenes (see [README.md](README.md)).

Key points:
- It is **not** an npm package with module imports.
- It is a set of global JS classes/functions loaded by `<script>` tags (for example [Linear Regression/part_1.html](Linear%20Regression/part_1.html)).
- Your scene code is a p5 instance sketch (`const Scene = function(s) { ... }; new p5(Scene);`).

Typical workflow:
1. Load `p5` (and optionally `p5.dom`, `katex`) in HTML.
2. Load Manim.js source files from `src/`.
3. In `preload`, load fonts/images/models.
4. In `setup`, call `setup2D(s)` or `setup3D(s)`, create objects.
5. In `draw`, call each object's `.show()` and trigger events by `s.frameCount`.

---

## 2) Setup, prerequisites, run model, and output

## Prerequisites
- Browser capable of running p5.js.
- Local files in this repo (no build step required).
- Optional libraries depending on scene:
  - p5 DOM API for KaTeX overlays (`lib/p5/p5.dom.min.js`)
  - KaTeX (`lib/katex/katex.min.js` + CSS)

## Include order (important)
Use the same style as existing pages (for example [Graph Algorithms/graph.html](Graph%20Algorithms/graph.html), [LLM Explained/DPO/dpo.html](LLM%20Explained/DPO/dpo.html)):

```html
<script src="../lib/p5/p5.min.js"></script>
<script src="../lib/p5/p5.dom.min.js"></script>
<script src="../lib/katex/katex.min.js"></script>
<link rel="stylesheet" href="../lib/katex/katex.min.css">

<script src="../src/graphics.js"></script>
<script src="../src/globals.js"></script>
<script src="../src/text.js"></script>
<script src="../src/timer.js"></script>
<script src="../src/utils.js"></script>
<script src="../src/brain.js"></script>
<script src="../src/figures.js"></script>
<script src="../src/3d.js"></script>
<script src="../src/math.js"></script>

<script src="your_scene.js"></script>
```

## Run examples
- Open existing HTML entry files directly in browser (for example `Linear Regression/part_1.html`).
- Rendering is real-time in canvas/DOM; there is no built-in offline frame renderer/export pipeline in this repo.

## Output behavior
- 2D: canvas from `setup2D` (`src/globals.js`).
- 3D: WEBGL + often offscreen composition (see `chapter_7.js` and `src/3d.js`).
- KaTeX text renders as DOM overlay positioned over canvas (`src/text.js`, class `Katex`).

---

## 3) Execution model and scene lifecycle

Standard skeleton:

```js
const MyScene = function(s) {
  let tnr, objs = {};

  s.preload = function() {
    tnr = s.loadFont('../lib/font/times.ttf');
  };

  s.setup = function() {
    setup2D(s); // or setup3D(s)
    objs.title = new TextWriteIn(s, { str: 'Demo', x: 100, y: 80, font: tnr, start: 10 });
  };

  s.draw = function() {
    // setup2D registers a pre-hook background; many scenes still call background explicitly
    s.background(0);

    if (s.frameCount === frames(2)) {
      // schedule transitions here
    }

    objs.title.show();
  };
};

new p5(MyScene);
```

Timing model:
- Global frame rate `fr` in `src/globals.js`.
- `frames(sec)` converts seconds to frame counts.
- Most classes use `start`, `end`, `duration` and internal timers (`Timer0/1/2` in `src/timer.js`).

Sequencing style:
- Construct objects once in `setup`.
- In `draw`, call `.show()` every frame.
- Trigger state changes with `if (s.frameCount === ...) object.move(...)`.

---

## 4) Core modules and API map (practical)

| Module | Purpose | Representative API |
|---|---|---|
| [`src/globals.js`](src/globals.js) | global settings/helpers | `frames(sec)`, `perFrame(v)`, `setup2D(s)`, `setup3D(s)`, `stdToP5`, `p5ToStd` |
| [`src/timer.js`](src/timer.js) | interpolation over frames | `Timer0`, `Timer1`, `Timer2`, `TimerFactory` |
| [`src/graphics.js`](src/graphics.js) | base classes | `PointBase` (move/shift/shake/jump), `Dragger` |
| [`src/text.js`](src/text.js) | text and math overlays | `Text`, `TextFade`, `TextWriteIn`, `Katex`, `AnimatedKatex`, `Citation` |
| [`src/utils.js`](src/utils.js) | geometry/plots/tables/images | `Axes`, `Grid`, `Line`, `Arrow`, `Circle`, `Arc`, `FcnPlot`, `Table`, `Pie`, `Rect`, ... |
| [`src/3d.js`](src/3d.js) | 3D objects/camera helpers | `Axes3D`, `Grid3D`, `Grid3D_Transform`, `Arrow3D`, `Plane3D` |
| [`src/figures.js`](src/figures.js) | special visuals | `AnimatedSVGPath`, `ImageFadeUp` |
| [`src/brain.js`](src/brain.js) | mascot/thought UI | `BrainBase`, `Bubble`, `ThoughtBrain`, `Emoji`, `Bulb` |

Graph-specific APIs are in [`Graph Algorithms/graph.js`](Graph%20Algorithms/graph.js): `Graph_U`, `Graph_D`, `Node`, `Edge`, `Tracer`.

---

## 5) Mathematical objects and primitives (actual availability)

Available directly:
- Points/markers: `Point`, `PlotPoint`
- Lines: `Line`, `LineCenter`, `DottedLine`, `TransmitLine`
- Vectors/arrows: `Arrow` (2D), `Arrow3D`
- Curves/circles/arcs: `Circle`, `Pie`, `Arc`, `ArcArrow`
- Coordinate helpers: `Axes`, `Grid`, `Axes3D`, `Grid3D*`
- Function plot: `FcnPlot`
- Regression plot helper: `Plot` (scatter + least-squares line)
- Tables/charts: `Table`, `Chart`
- Labels/text/math: `Text*`, `Katex`, `AnimatedKatex`
- Graph structures: `Graph_U`, `Graph_D`

Not provided as a first-class reusable primitive in `src/`:
- Generic polygon/triangle class for math geometry (draw via p5 directly if needed).
- Generic “Group” container class (use arrays/objects and iterate `.show()`).

---

## 6) Coordinate systems and transforms

## 2D canvas coordinates
- p5 convention: origin at top-left, +x right, +y down.
- `Axes` maps mathematical coordinates to canvas:
  - `canvasX = centerX + x * stepX`
  - `canvasY = centerY - y * stepY`
  (see `Plot.calcCoords` in `src/utils.js`).

## 3D conversion helpers
- `stdToP5([x,y,z]) -> [y,-z,x]` and inverse `p5ToStd` in `src/globals.js`.
- `Arrow3D` expects `from`/`to` in standard coordinates and converts internally.

## Camera/canvas
- `setup2D` / `setup3D` create canvas at `canvasWidth`/`canvasHeight` globals.
- `Axes3D` handles rotating camera and optional smooth camera movement (`moveCam`).

## Rotation/scaling
- 2D text rotation: `Katex` supports `rotation` style; many objects also use p5 transforms internally.
- KaTeX overlay auto-scales based on canvas DOM bounding rect (`src/text.js`).

---

## 7) Animation API patterns (timing, easing, sequencing)

Core principles:
- Every frame, objects update themselves in `.show()`.
- Timers provide normalized progress `t` in `[0,1]`.

Timers:
- `Timer0`: linear
- `Timer1`: decelerating
- `Timer2`: accelerate then decelerate
- `TimerFactory(frames, mode)` chooses one

Common animation methods:
- `PointBase.move(x, y, durationSeconds, timerMode)`
- `PointBase.shift(dx, dy, durationSeconds, timerMode)`
- `PointBase.shake(amp, durationSeconds)`, `jump(...)`
- `Line.move({x1,y1,x2,y2,duration,mode})`
- Color transitions via `StrokeChanger` / `FillChanger` and `reColor` methods in many classes.

Chaining/parallelism:
- Use `frameCount` checks to start independent animations at chosen frames.
- “Parallel” = call multiple object methods in same frame.

Custom animation strategy:
- Compose from `PointBase` + a timer, or wrap existing objects and trigger their methods.

---

## 8) Styling and appearance

Common style parameters across classes:
- `color: [r,g,b]` or `[r,g,b,a]`
- `strokeweight`
- `fill`
- `size` for text
- `mode` for text alignment (0 left-top, 1 center, 2 right-top, etc.)

Defaults are class-specific (examples):
- Global named colors in `src/globals.js` (`White`, `Red`, `Green`, `Blue`, `Yellow`, `Orange`).
- `Node` fill defaults to a darkened ring color (`vector_multiply`).

Z-order:
- Controlled by draw order in `draw()` and nested `.show()` calls.
- No dedicated z-index system for canvas objects.
- KaTeX is DOM overlay above canvas.

Fonts:
- Load in `preload` using `s.loadFont(...)`.
- Pass `font` into `Text*` classes.

---

## 9) Dynamic/updater patterns

There is no Manim-Python style updater registry. Pattern here is explicit frame logic:

```js
if (s.frameCount === frames(3)) obj.move(newX, newY, 1.2, 2);
if (s.frameCount === frames(4)) line.move({ x1: ..., y1: ..., x2: ..., y2: ..., duration: 1 });
obj.show();
```

Useful synchronization patterns:
- Keep references in a scene-level object and trigger all related objects together.
- For coupled labels/shapes, use wrapper classes (like `Node` internally owning `Circle` + `TextFade`).

---

## 10) Input/data handling for math visuals

Typical inputs:
- arrays of coordinates (`V`, `E`, sample points)
- functions (`FcnPlot.f`)
- symbolic strings (KaTeX text)

Example: data-driven graph:

```js
const G = {
  V: [[100,200],[260,100],[260,300],[420,200]],
  E: [[0,1],[0,2],[1,2],[1,3],[2,3]]
};
const g = new Graph_U(s, { V: G.V, E: G.E, font: tnr, start: 40 });
```

Recommendation for AI agents:
1. Parse the problem into typed structures (points, equations, ranges, labels).
2. Choose primitives and convert math coordinates via `Axes` mapping.
3. Schedule reveal/order with `frames(sec)`.

---

## 11) Reusable math-visualization patterns

- Equations/derivations: `Katex`, `AnimatedKatex`
- Function plots: `Axes` + `FcnPlot`
- Regression/statistics: `Plot` + `Table` + annotation
- Geometry constructions: `Point`/`Circle`/`Line`/`Arc`
- Vectors: `Arrow` (2D), `Arrow3D` (3D)
- Transformations: `Grid3D_Transform`, `Arrow3D.move`
- Probability slices: `Pie`/`Arc`
- Sequences/series displays: `Table` or repeated `TextWriteIn`

---

## 12) Debugging, errors, performance, determinism, pitfalls

## Debugging helpers
- `Dragger` displays live coordinates and allows dragging `PointBase` instances (`src/graphics.js`).
- `HelperGrid` draws scaffold grid (`src/utils.js`).
- `showFR(s)` displays current FPS (`src/globals.js`).

## Performance notes
- KaTeX is DOM-based; many nodes/updates can be expensive.
- 3D lighting/material in WEBGL is expensive (see comments in `Arrow3D.show`).
- Prefer creating objects once; avoid per-frame re-instantiation.

## Determinism
- Frame-based timing gives deterministic sequencing for fixed `fr`.
- Random helpers exist (`randomizeWeights`, `randomizeEdges` in graph module) and are non-deterministic unless you manage randomness externally.

## Common pitfalls
- Missing script include order causes `ReferenceError` (global dependency chain).
- `start`/`duration` units can vary by API (usually frames for `start/end`, seconds for `duration` then converted via `frames()`).
- Some APIs are incomplete/legacy-marked:
  - `Matrix` class placeholder and `QR_3` warning in `src/math.js`
  - `DottedLine` notes about limitations in comments
- No built-in export renderer; output is browser runtime.

---

## 13) AI-agent workflow section

## A) Plan from problem statement
1. Extract mathematical entities (functions, points, vectors, constraints).
2. Choose coordinate frame (`Axes`/`Grid` if needed).
3. Choose primitives for each entity.
4. Build an animation timeline in seconds -> convert with `frames()`.

## B) Translate to scene code
- Use one p5 sketch per deliverable.
- Instantiate all objects in `setup`.
- Put event triggers in `draw` using exact `frameCount` conditions.
- Keep object references in plain objects/arrays.

## C) Validate mathematical correctness
- Verify coordinate transforms (`centerX`, `centerY`, `stepX`, `stepY`).
- For arrows/vectors, ensure endpoint math before animation.
- For formulas, verify KaTeX source strings and symbols.

## D) Keep compatibility with this repo
- Use only APIs present in `src/` and established scene patterns.
- Do not assume ES modules or npm tooling.
- Use repo assets paths (`../lib/font/...`, images/models).

## E) Final artifact format
- Provide a runnable `.js` scene plus matching `.html` loader (or instructions on existing loader file).
- Include timing constants and readable named variables.

---

## 14) Compact API cheat sheet + do/don't

## Cheat sheet

- Setup:
  - `setup2D(s)`, `setup3D(s)`, `frames(sec)`, `perFrame(v)`
- Text:
  - `new Text(...)`, `new TextFade(...)`, `new TextWriteIn(...)`
  - `new Katex(...)`, `new AnimatedKatex({parts:[...]})`
- Geometry:
  - `new Line(...)`, `new Arrow(...)`, `new Circle(...)`, `new Arc(...)`
  - `new Rect(...)`, `new RoundedRect(...)`, `new LinedRect(...)`
- Plots:
  - `new Axes(...)`, `new Grid(...)`, `new FcnPlot({f, axes,...})`, `new Plot(...)`
- Data/table:
  - `new Table(...)`, `new Chart(...)`
- Graphs:
  - `new Graph_U(...)`, `new Graph_D(...)`
- 3D:
  - `new Axes3D(...)`, `new Arrow3D(...)`, `new Plane3D(...)`, `new Grid3D_Transform(...)`

## Do / Don’t

**Do**
- Use `frames(sec)` for timing constants.
- Use arrays/objects as manual groups.
- Call every object's `.show()` each frame.
- Keep path includes explicit and ordered.

**Don’t**
- Don’t invent module imports/exports.
- Don’t assume Python Manim API parity.
- Don’t recreate objects every frame unless intentional.
- Don’t rely on undocumented classes outside this repository.

---

## 15) Substantial runnable examples (10+)

> For all examples: use an HTML loader similar to existing files and include required `src/*.js` + libs.

### Example 1 — Basic primitives (line, arrow, circle, text)

```js
const Ex1 = function(s) {
  let tnr, line, arrow, circle, title;

  s.preload = function() {
    tnr = s.loadFont('../lib/font/times.ttf');
  };

  s.setup = function() {
    setup2D(s);
    line = new Line(s, { x1: 120, y1: 180, x2: 520, y2: 180, start: frames(0.3), color: Blue });
    arrow = new Arrow(s, { x1: 120, y1: 260, x2: 520, y2: 320, start: frames(0.8), color: Orange });
    circle = new Circle(s, { x: 760, y: 250, r: 170, start: frames(1.0), color: Yellow });
    title = new TextWriteIn(s, { str: 'Primitives', x: 90, y: 70, font: tnr, size: 42, start: frames(0.2) });
  };

  s.draw = function() {
    s.background(0);
    title.show(); line.show(); arrow.show(); circle.show();
  };
};
new p5(Ex1);
```

### Example 2 — Function plot with axes

```js
const Ex2 = function(s) {
  let axes, plot;
  s.setup = function() {
    setup2D(s);
    axes = new Axes(s, { centerX: 520, centerY: 350, stepX: 90, stepY: 90, labelX: 'x', labelY: 'y', start: frames(0.2) });
    plot = new FcnPlot(s, {
      axes,
      f: (x) => 0.2 * x * x - 1,
      start: frames(1.0),
      duration: 2,
      color: [77, 217, 77],
      strokeweight: 4
    });
  };
  s.draw = function() {
    s.background(0);
    axes.showAxes();
    plot.show();
  };
};
new p5(Ex2);
```

### Example 3 — Animated parameter/value (text + moving arrow)

```js
const Ex3 = function(s) {
  let tnr, a = 0, label, arr;
  s.preload = function() { tnr = s.loadFont('../lib/font/times.ttf'); };
  s.setup = function() {
    setup2D(s);
    label = new TextFade(s, { str: 'a = 0.00', x: 120, y: 80, font: tnr, size: 42, start: 1 });
    arr = new Arrow(s, { x1: 200, y1: 300, x2: 500, y2: 300, start: 1, color: Yellow });
  };
  s.draw = function() {
    s.background(0);
    a = Math.min(1, s.frameCount / frames(4));
    label.reset({ str: 'a = ' + a.toFixed(2) });
    if (s.frameCount === frames(0.5)) arr.move({ x2: 900, y2: 420, duration: 2, mode: 2 });
    label.show();
    arr.show();
  };
};
new p5(Ex3);
```

### Example 4 — Geometric construction (circle intersections + connector)

```js
const Ex4 = function(s) {
  let A, B, c1, c2, p1, p2, base;
  s.setup = function() {
    setup2D(s);
    A = [420, 360]; B = [760, 360];
    base = new Line(s, { x1: A[0], y1: A[1], x2: B[0], y2: B[1], start: frames(0.2), color: White });
    c1 = new Circle(s, { x: A[0], y: A[1], r: 320, start: frames(0.5), color: Blue });
    c2 = new Circle(s, { x: B[0], y: B[1], r: 320, start: frames(1.0), color: Green });
    p1 = new Point(s, { x: 590, y: 70, start: frames(1.5), color: Yellow });
    p2 = new Point(s, { x: 590, y: 650, start: frames(1.8), color: Yellow });
  };
  s.draw = function() {
    s.background(0);
    base.show(); c1.show(); c2.show(); p1.show(); p2.show();
  };
};
new p5(Ex4);
```

### Example 5 — Vector addition in 2D

```js
const Ex5 = function(s) {
  let u, v, sum;
  s.setup = function() {
    setup2D(s);
    const O = [250, 500], U = [620, 380], V = [470, 180];
    u = new Arrow(s, { x1: O[0], y1: O[1], x2: U[0], y2: U[1], start: frames(0.4), color: Blue });
    v = new Arrow(s, { x1: U[0], y1: U[1], x2: V[0], y2: V[1], start: frames(1.3), color: Green });
    sum = new Arrow(s, { x1: O[0], y1: O[1], x2: V[0], y2: V[1], start: frames(2.2), color: Orange });
  };
  s.draw = function() {
    s.background(0);
    u.show(); v.show(); sum.show();
  };
};
new p5(Ex5);
```

### Example 6 — Coordinate transformation (3D grid transform)

```js
const Ex6 = function(s) {
  let g3, axes, grid, ax;
  s.preload = function() {
    axes = s.loadModel('../lib/obj/axes.obj');
  };
  s.setup = function() {
    setup3D(s);
    g3 = s.createGraphics(canvasWidth, canvasHeight, s.WEBGL);
    const M = stdToP5([1, 0, 0, 0.5, 1, 0, 0, 0, 1]); // shearing-like matrix (converted)
    grid = new Grid3D_Transform(s, { mat: M, lineLen: 120, numLines: 3, start: frames(2) });
    ax = new Axes3D(s, { model: axes, speed: -0.0025 });
  };
  s.draw = function() {
    s.background(0);
    ax.show(g3);
    grid.show(g3);
    s.image(g3, 0, 0, canvasWidth, canvasHeight);
  };
};
new p5(Ex6);
```

### Example 7 — Least-squares scatter + fitted line

```js
const Ex7 = function(s) {
  let tnr, plot;
  s.preload = function() { tnr = s.loadFont('../lib/font/times.ttf'); };
  s.setup = function() {
    setup2D(s);
    plot = new Plot(s, {
      xs: [-2, -1, 0, 1, 2],
      ys: [-0.8, -0.2, 0.3, 1.0, 1.9],
      centerX: 560,
      centerY: 380,
      stepX: 130,
      stepY: 130,
      labelX: 'x',
      labelY: 'y',
      start: frames(0.4),
      startLSLine: frames(2.0),
      ptLabel: true,
      font: tnr
    });
  };
  s.draw = function() {
    s.background(0);
    plot.showAxes();
    plot.showPoints();
    plot.LSLine.show();
  };
};
new p5(Ex7);
```

### Example 8 — Directed weighted graph

```js
const Ex8 = function(s) {
  let tnr, g;
  const G = {
    V: [[180,240],[480,120],[480,360],[820,240]],
    E: [[0,1,0,5],[0,2,0,2],[1,3,0,3],[2,3,0,4]]
  };

  s.preload = function() { tnr = s.loadFont('../lib/font/times.ttf'); };
  s.setup = function() {
    setup2D(s);
    g = new Graph_D(s, { V: G.V, E: G.E, font: tnr, start: frames(0.6), color_v: Yellow, color_e: Green });
  };
  s.draw = function() {
    s.background(0);
    g.show();
  };
};
new p5(Ex8);
```

### Example 9 — KaTeX equation + animated parts

```js
const Ex9 = function(s) {
  let eq;
  s.setup = function() {
    setup2D(s);
    eq = new AnimatedKatex(s, {
      x: s.width / 2,
      y: 200,
      mode: 1,
      font_size: 44,
      parts: [
        { key: 'lhs', text: 'x^2', color: '#4dd999', start: frames(0.8) },
        { key: 'plus', text: '+', color: '#ebf1fa', start: frames(1.0) },
        { key: 'rhs', text: '1', color: '#ff7080', start: frames(1.2) }
      ]
    });
  };
  s.draw = function() {
    s.background(0);
    if (s.frameCount === frames(3)) eq.jumpPart('rhs', 25, 0.6);
    eq.show();
  };
};
new p5(Ex9);
```

### Example 10 — Probability-style pie sectors

```js
const Ex10 = function(s) {
  let a, b;
  s.setup = function() {
    setup2D(s);
    a = new Pie(s, {
      x: 430, y: 320, r: 320,
      a1: 0, a2: s.TWO_PI * 0.65,
      fill: [77, 177, 255, 180], color: [77, 177, 255], start: frames(0.4), duration: 1.4
    });
    b = new Pie(s, {
      x: 430, y: 320, r: 320,
      a1: s.TWO_PI * 0.65, a2: s.TWO_PI,
      fill: [255, 77, 97, 180], color: [255, 77, 97], start: frames(1.2), duration: 1.4
    });
  };
  s.draw = function() {
    s.background(0);
    a.show(); b.show();
  };
};
new p5(Ex10);
```

### Example 11 — Sequence table + explanatory text

```js
const Ex11 = function(s) {
  let tnr, table, txt;
  s.preload = function() { tnr = s.loadFont('../lib/font/times.ttf'); };
  s.setup = function() {
    setup2D(s);
    table = new Table(s, {
      font: tnr,
      x: 120,
      y: 220,
      xs: [1,2,3,4,5],
      ys: [1,4,9,16,25],
      label1: 'n',
      label2: 'n^2',
      start: frames(0.5),
      colorY: Yellow
    });
    txt = new TextWriteIn(s, {
      str: 'Sequence: a_n = n^2', x: 120, y: 130, font: tnr, size: 42, start: frames(0.2)
    });
  };
  s.draw = function() {
    s.background(0);
    txt.show();
    table.show();
  };
};
new p5(Ex11);
```

---

## 16) Repository-specific limitations and uncertainty notes

- The framework is actively evolved with mixed-era code; comments include `fixme/todo` and some legacy wording.
- `src/math.js` is not a polished linear algebra API; treat it as utility/experimental.
- Several advanced helpers are project-local (for example `LLM Explained/llm_anim.js` classes) rather than documented stable core APIs.
- If behavior is ambiguous, confirm by reading implementation and existing scene usage before generating code.
