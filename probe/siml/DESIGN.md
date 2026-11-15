# siml — the design

siml is the language a Tekton document can be written in. A siml file describes printable parts as
values, declares the parameters the inspector shows, and sets up, drives and queries simulations whose
solvers are Tekton's own. This chapter says what the language is, why each part of it is the way it is,
and what it is built from. It was written after the nex probe (`probe/nex/REPORT.md`), whose two
scripts are the specification by example: §12 rewrites both in siml.

The probe's verdict was that nex fits, given operators on user types and array concatenation. siml is
the other answer to the same evidence. It is not nex with fixes; it is a smaller language whose whole
shape follows from one requirement nex was never designed for: **a document that re-runs on every
slider tick, remembers which line made every face, and can be edited from the viewport.**

## 1. Purpose and non-goals

**What it is for.**

- Describing parts: solids, profiles, transforms, CSG, and selections of faces and edges.
- Declaring parameters, with their types, units, ranges and steps, which become the inspector's
  controls.
- Computing what a part's dimensions are derived from: volumes, clearances, a compression ratio
  turned into a deck height. The probe showed that this arithmetic is where units of measure pay.
- Setting up mechanisms (bodies, joints, drivers), running Tekton's native solvers over them,
  answering their events, and querying the results.

**What it is not.**

- **Not general-purpose.** It has no I/O library, no processes, no network, no clock, no randomness
  without an explicit seed, and no threads. A script reads a file only where the host grants one, and
  then only as an asset whose contents become part of the cache key (§5).
- **No async.** Nothing in a script waits. Long work (a mesh boolean, a solver run) is a host call
  that returns when it is done, on a worker thread the script never sees.
- **No solvers in the language.** The multibody solver, the meshing kernel (Manifold), contact
  detection and design-rule analysis are Tekton's, in sysl. A script describes what to solve and reads
  the answer. The probe's slowest runs (330 ms and 550 ms) were spent almost entirely in a stub solver
  written in nex; that time leaves the language altogether.
- **No acausal equations** at first (§7).
- **No FFI.** A script cannot reach C. Everything outside the language comes through host functions
  Tekton declares (§9).

## 2. The core model

**A script evaluates to a document.** The top-level `part` declarations are the document's parts, in
order, each named by its declaration. Tekton shows them, exports them and measures them. Nothing else
puts geometry on screen: there is no scene to mutate, no "current object", no `add_to_scene`.

```siml
part bottle = body()
part lid = screw_lid().move(x = 3 * radius)
```

**A solid is an immutable value.** `cylinder(10mm, 20mm)` does not mesh anything. It makes a node
holding an operation, its numeric arguments, its children and where it was made in the source. Nodes
are hash-consed: two equal operations on equal children share a structural hash, and that hash keys
Tekton's mesh cache. This is what Tekton's document already is ("a tree of operations, not a mesh ...
meshes are derived from it by the kernel and cached per node", `DESIGN.md` § The document). siml
produces that tree; it does not replace it.

**CSG is arithmetic.** `a + b` is the union, `a - b` the difference, `a * b` the intersection, and
`union(list)` folds a list. `-` binds like subtraction, so `cap - groove - holes + knurl` reads left to
right. The probe's largest gap ("`+` needs numbers; found V and V") is closed by the language rather
than by operator overloading in general: siml has no user-defined operators. The prelude defines
these three on `Solid` and on `Profile`, and on numbers, vectors and lists (`++` concatenates lists),
and that is all.

**Transforms are methods.** Any function may be called with its first argument before a dot, so
`cylinder(r, h).move(z = -h / 2).turn(X, 90deg)` reads in the order the operations happen. `move`,
`turn`, `scale`, `mirror`, `polar` (n copies about Z) and `array` (n copies along a vector) are
ordinary prelude functions. A primitive stands on the XY plane, centred on Z, as a shape dropped from
Tekton's palette does; `center = true` centres it on the origin.

**Profiles are values too.** `circle(r)`, `ellipse(rx, ry)`, `rect(w, d)`, `polygon(points)` and a path
builder (`path(p).line_to(q).arc_to(r, through = s).close()`) make `Profile`s, which have the same
`+ - *` and the transforms in the plane. `extrude`, `revolve`, `loft` and `sweep` turn them into
solids. A constrained sketch, a profile solved from constraints rather than placed, is later
(§13, question 5).

**Selectors take faces and edges by what they are, never by an id.** An operation that needs part of
a solid's boundary takes a predicate:

```siml
outer.shell(wall, open = top)
body().fillet(e -> e.circular and e.center.z < 1mm, 3mm)
```

`Face` and `Edge` are records the kernel answers for an evaluated solid: kind (`planar`,
`cylindrical`, `circular`, `straight`), `normal`, `center`, `area`, `length`, and the solid's box.
`top`, `bottom`, `facing(v)`, `parallel(v)` and `at_max(axis)` are prelude predicates written in siml.
This is CadQuery's and build123d's lesson (`.faces(">Z")`), taken without the string mini-language. A
selector is re-run on every evaluation, so there is no topological-naming problem to solve: no face id
ever outlives the run that answered it. The price is that a selector forces the kernel to mesh its
solid during the script's run, and that call is memoised like any other (§5).

**Values are values.** A list is `[T]`, zero-based (`xs[0]`, `0..<n`), immutable, and updated in
place only where its reference count is one, so a local `var p = fill(720, 1bar)` written element by
element in a loop is as fast as an array. A struct or an enum is a sysl-shaped declaration with
fields; `match` is postfix, as in slate. A function is a value; a lambda is `x -> e`, and a
two-parameter lambda given where a function of one pair is expected takes the pair apart
(`pairs.map((below, width) -> …)`). `%` is floored, so `(i - 540) % 720` is an index. A local `var` is
allowed and invisible from outside the function, so it costs purity nothing; a module-level `var` does
not exist (a `param` is the only module-level binding the host may change).

## 3. Parameters

```siml
/// The bottle's overall height, base to the top of the neck's shoulder.
param height = 150mm in 80mm..220mm by 1mm
param waist = 0.82 in 0.5..1.0 by 0.01
param cylinders = 1 in [1, 4]
param style = Round in EdgeStyle
param vented = true
```

A `param` is a module-level binding whose value the host may set. Its declaration carries everything
the inspector needs, so there is no second list of sliders to keep in step (the probe's `sliders()`
had to be): the default, the type and unit (from the default: `150mm` is a length shown in mm), a
range (`in lo..hi`), a step (`by s`), a choice (`in [1, 4]`, or `in` an enum), and the `///` comment
above it as the tooltip. A `bool` is a checkbox. A default written `6in` shows in inches while the
script computes in whatever its expressions need; the display unit is the literal's.

**The host reads them as data**: `script.params()` answers, for each, its name, kind (quantity, int,
bool, choice), dimension, display unit, default, range, step, doc text and the span of its default
literal. **The host sets them** in the display unit (`script.set("height", 160, mm)`), and the value
is checked against the range and step before the run; an out-of-range set is refused with the range in
the message, so a file edited by hand cannot put a slider past its end. Every `param` is a dependency
root for the cache (§5): changing one invalidates exactly the calls that read it.

**A unit is a value** (`mm`, `bar/deg`) of type `Unit`, so the parameter API needs no stringly-typed
unit and a library may take a display unit as an argument. This closes the probe's fourth missing
feature, where `Slider` stripped the number and spelled its unit as text.

## 4. Two-way editing

**Every solid remembers where it was made.** A node records the span of the call that made it and the
chain of calls, in the user's own files, that led to that call: `polar` at line 85, inside
`screw_lid()`, called from `part lid` at line 92. The chain is a shared persistent list, so recording
it costs one pointer per node. A memoised call (§5) answers the solid with the chain from inside the
call, and the caller wraps it in a provenance-only node carrying its own call site, which costs no
kernel work and keeps the chain right when one cached result is used from two places.

**From the viewport to the code.** Tekton already resolves a click to a document node. siml needs
that resolution to survive booleans, and Manifold provides it: each primitive is made with
`as_original()`, whose id its triangles keep through every union, difference and intersection
(`sysl-lang/manifold` exposes `as_original` and `original_id`; the per-triangle run table may need
adding to the binding). So a click on the inside of a vent hole lands on the hole's primitive, not on
the lid it was cut from, and the editor jumps to the hole's line. A second click walks one step
outward along the chain, to the caller.

**From the code to the viewport.** Selecting text highlights every node whose chain contains a span
inside the selection: select `screw_lid()`'s body and the whole lid lights; select one line of it and
only what that line made lights. Selecting a `param`'s name highlights the geometry whose evaluation
read it, which is the cache's own dependency record (§5) read backwards.

**From the viewport back into the code.** Dragging a handle rewrites text, and a rewrite is offered
only where the next run is guaranteed to put the geometry where the drag left it. The rule is
syntactic, decided when the script is compiled, not traced at run time. For a transform's numeric
argument, or a primitive's size argument:

1. **The argument is a literal** (`move(x = 30mm)`): the literal is rewritten (`32mm`), in its own
   unit and to the drag's snap, keeping everything around it byte for byte.
2. **The argument is `E + literal` or `E - literal`**: that literal is rewritten.
3. **The argument is any other expression** (`x = 3 * radius`): ` + 2mm` is appended to it, which
   makes it case 2 for the next drag. The parameter `radius` is never changed by a drag, because
   `radius` also sizes the bottle.
4. **The part has no transform of that kind**: `.move(x = 2mm)` (or `.turn(Z, 15deg)`) is appended to
   the part's expression.

Only the argument's own span is ever touched, never a `val` it reads: rewriting `val gap = 10mm`
because one of its five uses was dragged would move the other four. A rotation is offered only about
the transform's own axis; a free-orbit rotation has no single number to write.

**Which edits are reversible, and which are not.** Every edit above is reversible in the plain sense:
it is one text change, it goes on the editor's undo history (the one history; the document has no
second one), and the run after it reproduces the dragged state exactly. Setting a `param` from the
inspector is reversible in the same way, and writes its default literal only on Save to Source. These
are refused, with a message naming the line to edit instead:

- **A node made more than once by one span**: one of the six holes `polar` made, one ring of a
  `map`. Moving one alone has no text. Tekton offers to move all of them, through the shared span
  (rules 1 to 3 apply to it), and shows them moving together.
- **A node made inside a library**, a prelude function or an imported file the user did not open: the
  drag is taken back to the user's own call site, and applies there if rules 1 to 4 do.
- **A value that came out of a query or a simulation** (placed at `bbox(s).hi.z`, at a
  `position_of`): its number is an answer, not an input.
- **Geometry edits that are not numbers**: pushing a face, moving a vertex, reshaping a loft section.
  siml has no text for them, and inventing one (a recorded "face offset" operation) is the
  modifier-stack fragility Tekton's own design rejects.

## 5. Incremental re-evaluation

A slider tick must re-run only what depends on the slider. That is sound only if the language makes
two promises, and siml makes them by construction.

**Purity.** A function's result is determined by its arguments and by the module-level bindings it
reads, through everything it calls. There is no module-level mutable state; a local `var` is local;
a closure passed in is an argument, hashed as its function plus its captured values; and a host
function is declared `pure` or it makes every caller unmemoisable (Tekton declares none that is not).
The elaborator proves the read set: nex's `effects.sysl` computes exactly this, "the module-level
`var`s it reads and writes, the I/O it does ... transitive ... by iterating to a fixed point over the
call graph", and siml takes it.

**Determinism.** The same inputs give the same bits. Numbers are IEEE `real64` with no fast-math and
no reassociation; iteration over every collection is in insertion order; there is no clock and no
unseeded randomness (`random(seed)` answers a generator value). The kernel must also be deterministic
for one input, which Tekton already relies on (a document exported twice is the same bytes).

**The effects that remain** are part of the value. `print` writes to Tekton's console; `warn` and
`check` attach a diagnostic to a span (§8). Each is recorded with the call that made it, and a cache
hit replays the record, so a run served from the cache prints and warns exactly as a cold run does.

**What is cached, and by what key.**

- **Module-level calls.** A call of a top-level function or the evaluation of a module-level `val` is
  keyed by (the function, the structural hash of each argument, the hash of each `param` and
  module-level `val` in its read set). Its entry holds the result, the effect record and the read
  set. Lambdas and local functions are not keyed: they are cheap, and their callers are.
- **Kernel results.** A mesh is keyed by its node's structural hash, which excludes provenance. This
  is Tekton's existing per-node mesh cache, now shared by two scripts' worth of identical holes.
- **Selector answers, queries and simulations** are host calls declared `pure`, keyed like any call.

A value's hash is structural and computed once, at construction: a real by its bits (one NaN), a list
by its elements, a solid by its operation, arguments and children's hashes, a closure by its function
and captures. Entries carry the generation they were last used in; those not touched for a few runs,
or past a byte cap, are dropped. An entry is written only when its call returns, so a cancelled run
(§9) leaves the cache consistent and has still warmed it. A drag that is abandoned halfway has made
the next frame cheaper, not wrong.

**What this buys in the shaker.** `radius` is read by `section_r`, which `volume_below`, `height_for`
and `body()` read; `screw_lid()` reads `pitch`, `clearance` and constants, not `radius`. Dragging
`radius` re-runs the bottle's functions and the lid's `move`, and the lid's mesh, with its 37 cuts and
36 knurls, comes from the cache unchanged.

## 6. Units of measure

siml takes nex's units whole: the exact rational factors, the namespace of their own, the `10mm` and
`9.81<m/s^2>` spellings, conversion where two scales meet (`promote` and `TRescale`), and diagnostics
that name both dimensions. They kept every quantity of the probe's Otto model right "with no hand
conversions" and are the reason the probe's engine script is believable. The type position spells a
unit directly: `h: mm`, `p: bar`, `rate: bar/deg`, `r_air: J/(kg K)`; `real` is dimensionless.

The probe's frictions are fixed:

- **A number in a unit expression is a factor.** `unit rpm = turn / 60 s` is accepted, and the
  prelude declares it, beside `turn = 360 deg`, `bar = 100000 Pa`, `ml = cm^3` and `l = 1000 ml`.
- **An angle stays inside a rate.** `2bar / 1deg` is `2 bar/deg`, not `114.59 bar`. Radians drop out
  only where a declared type asks for it, and only radians: a torque `N m` from a force times an arm
  in `mm/rad`, a power `kW` from a torque times `3000rpm` (the turn becomes 2π rad and the rad drops,
  so the inline four's power needs no `/ 60` and no `2pi`). Everywhere else angle is a dimension like
  any other, and a mismatch is reported.
- **A unit is a value** (§3), so a struct generic over a dimension works:
  `struct Port[D: Dim]`, `Speed[D](port: Port[D], rate: D / s)`. This is nex's g05, "a unit
  parameter is not elaborated yet", done.
- **`unit` is a soft keyword**, a declaration's first word only, so a field may be called `unit`.
- **`floor`, `ceil` and `round` of a `real` answer an `int`**, so `1..floor(capacity / 100ml)` is a
  range. A quantity divided by a quantity of its own dimension is a `real`, which is how a count of
  marks comes out of two volumes.
- **`0` and `inf` take their unit from context**: `var v = 0` as `ml` where the declared or inferred
  type says so, `var gap: mm = inf`. Every other literal states its unit.
- **Formatting knows units**: `"${capacity:.0f}"` prints `380 ml`, `"${capacity as l:.2f}"` prints
  `0.38 l`, and `x / 1mm` is the honest way to a bare number. nex's `strip()` is gone.

## 7. Simulation

**Mechanisms are values.** A `Body` is a solid with a mass (and, optionally, an inertia the kernel
computes from it by default) and whether it is fixed. `body.at(point)` is a `Frame` on it. A joint
joins two frames: `Revolute(a, b, axis = Y)`, `Prismatic(a, b, axis = Z)`, `Fixed(a, b)`. A
`Mechanism` is the bodies, the joints and a driver. Bodies are referred to by value, not by name; a
mechanism refuses two bodies of one name, because the name is what the viewport and the console show.

**Ports are typed by dimension.** Modelica's connector idea, taken causally: a joint exposes the
quantities across it, each a `Port[D]`. A revolute has `angle: Port[Angle]` and `torque:
Port[Torque]`; a prismatic has `position: Port[Length]` and `force: Port[Force]`. A driver connects to
a port and is checked against it: `Speed(main.angle, 3000rpm)` type-checks because `rpm` is an angle
per time, and `Speed(bore.position, 3000rpm)` is refused with both dimensions named. A state answers a
port's value as a quantity of that dimension: `st[main.angle]` is a `deg`.

**Running is a call with handlers.** `simulate` takes the mechanism, the port to step, the range and
the step, and an indented block holding the run's state, its handlers and its answer:

```siml
simulate(e.mech, e.main.angle, 0deg..720deg, step = 1deg)
    var worst = 0mm
    on step(st)
        worst = max(worst, ...)
    on contact(st, a, b)
        warn("${a.name} meets ${b.name} at ${st[e.main.angle]}")
    worst
```

The native solver calls `on step` after each step and `on contact` when two bodies touch, which is
ExtendSim's ModL message-handler model: a block's code is what runs when a message arrives, not a loop
the modeller writes. The block's `var`s are the run's state; the handlers are closures over the
block's frame; the final expression is the answer after the run. Semantically it is a fold, so the
whole `simulate` is a pure function of its inputs and is memoised like any call. A handler may
`stop(reason)` the run early.

**Queries** are host functions over a state: `st.position(body)`, `st.pose(body)`,
`st.angle(body)`, `st.point(body, local)`, `st.min_gap(a, b)`, `st.overlaps(a, b)`, `st.reaction(joint)`.
The probe's script-driven loop (`st = step(st, 1deg)` in a `for`) remains possible, since `solve` and
`step` are ordinary pure host functions, but handlers are the intended form: one native loop, with no
state value copied per step.

**Studies and the viewport.** A `part` may be a mechanism; Tekton shows its bodies posed at the
driver's start and gives the viewport a scrubber over the driven port, which calls the solver
directly and runs no script. A `study` is a named computation run on demand from a Studies panel
under the simulation budget (1 s), not on every slider tick; its answer is shown by type: a struct as
a table of named quantities, a `[bar]` as a plot over its index, a list of structs as a table with a
row each. A `report` is a cheap computation shown live in the inspector.

**Acausal equations are out of scope at first.** A Modelica-style model is a set of equations in
which the direction of each connection is not stated. Running one needs symbolic index reduction
(Pantelides), tearing, sorting into blocks, and an implicit DAE integrator. That is a compiler and a
numerical library larger than the rest of siml together, and its failures are the hardest in the
field to diagnose for a user ("structurally singular system"). The mechanisms a 3D printing user
models — linkages, cams, gears, a piston — are multibody problems with a fixed causal structure, which
a native solver handles without any of it. If a real need appears (a hot end's thermal network, a
stepper's electrical side), the first step is a native solver for one domain's networks with typed
ports, not general acausality.

## 8. Diagnostics

Every diagnostic has a span in a source file and appears three ways: underlined in the editor, listed
in the console, and, where it concerns geometry, as a marker on the part in the viewport.

- **Static** (parsing and elaboration): nex's quality is the target and its messages are the
  starting point: "argument `z` must be `mm`, but this is `deg`: angle and length are different
  dimensions", "`cylinder` needs its argument `h`", "`shell` has no parameter `opne`; did you mean
  `open`?" (`nearest_name` in `sh.sysl.parsing.diag`, which nex's elaborator already imports). Every
  error in a file is reported in one pass. A shadowing that breaks a call says so: "`step` is the
  loop variable at line 210 here, an `int`, which hides the function `step`".
- **Run-time traps** (an index out of range, `sqrt` of a negative length, a `loft` given three
  heights for four sections): the message and the siml stack, each frame a span. The previous good
  geometry stays on screen, dimmed, and the trapping line is marked. The probe's `nex run` printed a
  trap with no location; siml has no path that does.
- **Checks**, written in the script, are warnings, not traps, so a part that breaks a rule is still
  shown and can be fixed by dragging:

  ```siml
  check wall >= 2 * nozzle else "wall ${wall}, at least two nozzle widths (${2 * nozzle})"
  ```

  A `check` inside a function returning a `Solid` is also attached to the solid, so its marker sits
  on the part.
- **Design rules**, computed by Tekton natively on the final mesh against the printer profile it
  already holds (a 0.4 mm nozzle on a Prusa CORE One): minimum wall, minimum hole, unsupported
  overhang, a part outside the build volume. Each finding is a region of triangles; Manifold's
  original ids take it to the primitive, and the primitive's chain to the line:
  **"wall 0.3 mm here, minimum 0.8 mm (0.4 mm nozzle)"**, shown on the thin region and at the `shell`
  call that made it, with the `wall` parameter named when the span's arguments read it.

## 9. Embedding

Tekton hosts siml in-process, in sysl, through an API shaped like nex's `embed.sysl` (`Host`,
`give`, `give_type`, `load_script`, `Script.call`, `CancelToken`), with four changes the probe asked
for.

**Host declarations are siml, so defaults and closures work.** Tekton ships `tekton.siml`, a prelude
whose host functions are declared, not defined:

```siml
host type Mesh
host pure cylinder_mesh(r: mm, h: mm, segments: int) -> Mesh
host pure faces(s: Solid) -> [Face]
host pure fillet(s: Solid, where: Edge -> bool, r: mm) -> Solid
host pure simulate[T](m: Mechanism, port: Port[Angle], over: Range[deg], step: deg = 1deg,
                      handlers: Handlers[T]) -> T
```

A default is a siml expression, filled in at the call site by the elaborator before the host is
called (nex's `tir_subst` already does this for its own `def`s), so the host receives every argument
and the probe's two-layer API of externs plus wrappers is unnecessary. A function-typed parameter is
handed to the host as a `SimlFn` handle, which the host calls back through `script.invoke(f, args)`
on the same thread, re-entrantly, under the same budget. nex refuses both today (`gaps/g08`).

**Opaque host types.** `host type Mesh` is nex's `@host struct`: no fields, compared by identity,
reference counted, released through the host's `release` when the last holder lets go. `Solid` is not
a host type: it is a siml value (§2) so that it can be hashed, cached and given provenance without a
kernel call. Only what the kernel computes (a mesh, a face list) is opaque.

**Values in the units the declaration names.** A host function declared `r: mm` is handed millimetres
however the script spelled the length, as in nex (DESIGN §16.3). A quantity going to Tekton is the
number in its declared unit; a parameter set from the inspector comes in its display unit.

**Cancellation and budgets.** A live re-run happens on a worker thread. A slider tick cancels the run
in progress through its `CancelToken` (one atomic word, read at every call entry and loop pass,
nex's design) and starts the next. Two limits apply. A **step budget** is deterministic, and the tests
use it. A **wall-clock deadline** (50 ms for parts, 1 s for a study) only decides what the user sees:
past it, Tekton keeps the last complete result on screen, dimmed, and lets the run continue to a hard
limit (10 s), after which it halts with the stack of where it was. A deadline never changes a result.
A run is either complete or discarded.

**The sandbox.** Every script is sandboxed; there is no unsandboxed mode to forget to switch on. No
FFI, no `stop` of the host, no file but a granted asset (`asset("knob.stl")`, its contents hashed into
every key that reads it), a cap on recursion depth and on allocation, and the step budget. A siml file
from a stranger is as safe to open as an STL.

**The result crosses as a frozen value.** The document a run answers (parts, reports, diagnostics,
the provenance table) is frozen (`buf.freeze()`, crossing as `&sync Frozen[T]`) and handed to the UI
thread, which never sees a value still being built.

## 10. Implementation and reuse

### The execution strategy

**siml compiles its typed IR into sysl closures and runs them: nex's interpreter strategy, not slate's
bytecode VM.** The evidence:

- **The probe's parts ran in 10–12 ms** on nex's closure-compiled interpreter, *including* loading and
  elaborating the file, against a 50 ms budget. With memoisation (§5) a slider tick re-runs a fraction
  of that.
- **The simulation runs (330 ms, 550 ms) were the stub solver**, written in nex and run by the
  interpreter. In Tekton that loop is native. What remains per step is a handler doing a few queries
  and comparisons, microseconds against a 1 s budget for 720 steps.
- **The Otto cycle, 30 ms**, is the most script-bound workload in either file (about 6,000 calls of
  `dpdth`), and is inside budget too.
- **slate's VM exists for one reason, and siml does not have it.** slate's README: "`await` has to
  suspend in the middle of an expression, and a tree-walker's state is the host language's own call
  stack, which cannot be captured." siml has no `await`. Its handlers are ordinary re-entrant calls
  from the host, which a closure interpreter makes on the sysl stack.
- **Static types make closure compilation pay**: the operation for each node is chosen once, at
  compile time ("a closure is nothing but the work", `interp.sysl`). slate's inline caches
  (`site_cache.sysl`) and fused instructions (`run_fused.sysl`) are answers to dynamic typing.

A bytecode form stays possible if a profile of real scripts ever shows dispatch dominating. Nothing in
the design depends on the choice.

**Memory: reference counting, not a tracing collector.** slate traces because "slate makes a cycle on
every named function: binding a closure into the very scope that closure captured". siml cannot make
that cycle: values are immutable, a closure captures values (the `simulate` block's handlers capture a
frame, and a frame's `var`s may not hold functions), and a local recursive function refers to itself
by its number, not through its environment. With no cycles, sysl's own `&T` reference counting frees
everything promptly, gives the in-place update of a uniquely held list, and keeps host objects'
`release` deterministic. `sh.sysl.gc` stays the fallback if a later feature creates cycles.

### File by file

**`sysl-lang/parsing`: used as it is.** `source.sysl` (spans, the line table), `scan.sysl`,
`classes.sysl`, `literal.sysl` (reading and `write_quoted`), `layout.sysl` (indentation as structure,
which nex and slate both use), `tokens.sysl`, `scope.sysl`, `pratt.sysl` (binding power), `diag.sysl`
(carets, `nearest_name`).

**Goes to shared packages first**, because it mentions no language's tokens or tree:

- **Source edits → `parsing/source.sysl`**: an edit list (span, replacement text), applied to a
  `Source` with every later span remapped. Two-way editing (§4) and any formatter need it.
- **Units → a new `sysl-lang/units` package**: `units.sysl` and the `Ratio`/`Unit` types from
  nex's `types.sysl` (exact rationals in `i256`, dimensions, factors with a power of π, angle
  handling, `same_dims`, `unit_eq`, `subst_unit`). It is not parsing machinery, so it is its own
  package, and nex and siml both consume it rather than siml carrying a copy.
- **The shortest round-tripping number writer**: slate's `decimal.sysl` ("the exact decimal
  arithmetic all three of the number writers stand on") belongs beside `write_quoted` in
  `parsing/literal.sysl`. A rewritten literal (`37.5mm`, not `37.500000001mm`) depends on it.

**From nex** (`sh/sysl/nex/`):

| file | decision | why |
|---|---|---|
| `lex.sysl` | adapt | the unit-literal rules (`10mm`, `<…>` after an operand) and interpolation are what siml needs; drop complex `i`, bit widths, Fortran spellings |
| `parse.sysl` | adapt, heavily | keep the unit-expression, literal and type sub-grammars; the declaration grammar is new (`name(…) -> T`, `param`, `part`, `report`, `study`, `check`, `host`, `simulate` blocks, `on`, postfix `match`) |
| `ast.sysl` | adapt | the same tree shape, new declaration kinds |
| `elab.sysl`, `elab_expr.sysl`, `elab_call.sysl` | adapt | names, calls, named arguments and defaults are right; lift the one-argument `print` restriction |
| `elab_units.sysl` | take, change two rules | the angle-drop rule (§6) and numbers inside unit expressions |
| `elab_aggregate.sysl` | take | structs, enums, tuples, `match`, destructuring, capture rules |
| `elab_generic.sysl` | adapt | add dimension parameters (`[D: Dim]`) |
| `elab_extern.sysl` | adapt into `host` declarations | allow defaults and function-typed parameters, add `pure` |
| `elab_module.sysl`, `modules.sysl` | adapt | a script may import a library file of its own |
| `elab_text.sysl` | take the interpolation half | format specs gain units |
| `effects.sysl` | take | the transitive read set is the memo key (§5) and the proof of purity |
| `tir.sysl`, `tir_subst.sysl`, `tir_order.sysl`, `tir_erase.sysl` | take | the typed IR, defaults filled at the call, module-level initialization order, units erased to rescales |
| `interp.sysl`, `interp_value.sysl`, `interp_builtin.sysl`, `interp_text.sysl`, `interp_units.sysl` | adapt | the closure compiler and its values; add hashing, provenance chains and the memo table at call entry |
| `embed.sysl` | adapt | `Host`, `HostType`, `NexValue` crossing, `CancelToken`, budget; add `SimlFn` and `invoke`, params as data |
| `show.sysl`, `show_tir.sysl` | take | the printed forms the tests compare |
| `elab_array.sysl`, `elab_prelude.sysl`, `interp_array*.sysl`, `interp_store.sysl` | leave | rank-N column-major arrays, fusion and assumed rank are Fortran's model; siml's lists and `Vec` are far smaller |
| `elab_parallel.sysl`, `parallel_check.sysl`, `elab_mpi.sysl`, `elab_lapack.sysl`, `interp_la.sysl`, `interp_kernels*.sysl`, `tir_gemm.sysl`, `tir_fuse.sysl`, `tir_parallel.sysl`, `inout_check.sysl`, `elab_bits.sysl`, `elab_pack.sysl`, `elab_table.sysl`, `elab_file.sysl`, `elab_view.sysl`, `literate.sysl` | leave | numerics, parallelism, `inout`, files and tables siml does not have |
| `compiler/`, `runtime/` | leave | no MLIR, no AOT; the interpreter's printed words come from siml's own small runtime |

**From slate** (`dev/slatelang/slate/`):

| file | decision | why |
|---|---|---|
| `pattern.sysl`, `ast_pattern.sysl` | adapt | the pattern grammar (literals, shapes, `|`, `@` bindings, guards) and its postfix `match` |
| `match.sysl` | adapt | the matching semantics, onto typed values |
| `check_cover.sysl` | adapt | whether a `match` over a data type leaves a variant unmatched; siml makes it an error |
| `decimal.sysl` | move to `parsing` | above |
| `lex.sysl` | leave | it is slate's own token set over the same `layout` pass nex's lexer uses |
| `vm.sysl`, `code.sysl`, `unit.sysl`, `compile*.sysl`, `run_fused.sysl`, `slots.sysl`, `site_cache.sysl` | leave | the VM is for `await`; caches and slots are for dynamic typing |
| `obj.sysl`, `collector.sysl` (over `sh.sysl.gc`) | leave | no cycles, so no tracing (above) |
| `check*.sysl`, `result.sysl`, `flow.sysl` | leave | slate's gradual checker; siml is statically typed by the nex elaborator |
| `embed*.sysl` | leave | a C surface; Tekton is sysl and hosts siml in-process |

Two principles are taken from slate's `CLAUDE.md` rather than code: a diagnostic "is judged by where it
sends the reader", and a module stays under a thousand lines with its seams chosen by counting
crossings.

**New in siml**: the `param`/`part`/`report`/`study` declarations and the document they make; the
solid node (hash-consed, with provenance); the memo table; the drag rules of §4; the mechanism, port
and `simulate` prelude; and `tekton.siml`, the host prelude. Tekton gains the host side: the kernel
calls, the selector answers, the solver loop that calls handlers, and the design-rule pass.

## 11. Prior art

- **OpenSCAD.** Takes: a model is a program whose value is the geometry; CSG as the vocabulary;
  parameters a customizer reads from comments. Avoids: values with no queries (no "how big is this",
  no faces to select, so every dimension is recomputed by hand), no units, and whole-model re-renders.
- **CadQuery / build123d.** Takes: selectors, which are what make fillets, shells and workplanes
  expressible without ids. Avoids: Python's baggage (a general-purpose runtime, an install, mutation
  through context managers, no units, no sandbox), and the selector string language (`">Z"`), replaced
  by typed predicates.
- **Blender `bpy`.** Takes nothing but the lesson: a script that mutates a live scene through an
  operator API depends on the scene's state (selection, mode, active object) and breaks when any of it
  differs. siml answers a document; it never edits one.
- **ExtendSim ModL.** Takes: message handlers (`on step`, `on contact`) as the shape of simulation
  code; variables bound to the interface (siml's `param` is the declaration-side form of ModL's dialog
  variables); hierarchy (a mechanism of bodies). Avoids: the dated C subset, global mutable block
  state, and handlers that can run in an order the model does not state.
- **Modelica.** Takes: connectors as typed ports, units checked through every equation, and the idea
  that a component's interface is its physical quantities. Avoids, for now: acausal equations and the
  symbolic machinery under them (§7), and a library ecosystem whose weight is the price of generality
  siml does not need.

## 12. The probe's scripts in siml

These are complete. They compute what the nex versions compute, written the way siml means them to be
written. The numbers in `probe/nex/REPORT.md` (380 ml; 1.981 mm at TDC; IMEP 11.5 bar; 183 N·m mean
for the inline four) are the oracle a first siml must reproduce. Two deliberate differences from the
nex versions: the graduation rings hug the body (the probe's were solid discs, which would have filled
the bottle), and the console printing of `main()` becomes `report`s and `study`s, which is how a
document shows numbers.

### `shaker.siml`

```siml
// A printable shaker bottle: a lofted, waisted body hollowed to a thin wall, graduated every
// 100 ml, with an external thread on its neck, and a screw lid with an internal thread, a
// clearance and a pattern of holes in its top.

/// The bottle's height, base to the top of the shoulder.
param height = 150mm in 80mm..220mm by 1mm
/// The base's outer radius.
param radius = 34mm in 20mm..50mm by 0.5mm
/// The waist's radius as a share of the base's.
param waist = 0.82 in 0.5..1.0 by 0.01
param wall = 1.6mm in 0.8mm..3mm by 0.1mm
param pitch = 3mm in 1.5mm..4mm by 0.25mm
/// Between the lid's thread and the neck's, on every side.
param clearance = 0.35mm in 0.1mm..0.8mm by 0.05mm

val neck_r = 16mm
val neck_h = 14mm
val lid_wall = 2mm
val hole_d = 2.5mm
val nozzle = 0.4mm

// ISO's basic thread profile stands 0.6134 pitch proud of the root on each side.
val major = 2 * neck_r + 2 * 0.6134 * pitch

// The body's outer sections, bottom to top: base, waist, shoulder, top.
val section_z = [0mm, 0.45 * height, 0.8 * height, height]
val section_r = [radius, waist * radius, 0.97 * radius, neck_r + 4mm]

// The outer radius at height `z`, the loft taken as straight between each pair of sections.
radius_at(z: mm) -> mm
    var k = 0
    while k < section_z.len - 2 and z > section_z[k + 1]
        k += 1
    val t = (z - section_z[k]) / (section_z[k + 1] - section_z[k])
    section_r[k] + (section_r[k + 1] - section_r[k]) * t

// The inside's volume up to height `z`: a frustum for each slice, less the wall.
volume_below(z: mm) -> ml
    var v: ml = 0
    for k in 0..<section_z.len - 1
        val z0 = section_z[k]
        val z1 = min(section_z[k + 1], z)
        if z1 > z0
            val r0 = radius_at(z0) - wall
            val r1 = radius_at(z1) - wall
            v += pi * (z1 - z0) * (r0^2 + r0 * r1 + r1^2) / 3
    v

// The height at which the inside holds `v`, by bisection.
height_for(v: ml) -> mm
    var lo = wall
    var hi = height
    for _ in 0..<50
        val mid = (lo + hi) / 2
        if volume_below(mid) < v then lo = mid else hi = mid
    (lo + hi) / 2

val capacity = volume_below(height)
val marks = 1..floor(capacity / 100ml)

body() -> Solid
    // The waist is an ellipse, flattened for the grip; the other sections are round.
    val sections = [circle(section_r[0]), ellipse(section_r[1], 0.9 * section_r[1]),
                    circle(section_r[2]), circle(section_r[3])]
    val outer = loft(sections, section_z)
    val hollow = outer.shell(wall, open = top)
    val neck = tube(neck_r, neck_r - wall, neck_h).move(z = height)
    val ridge = thread(major, pitch, neck_h - 2mm).move(z = height + 1mm)
    // A raised ring at every 100 ml, standing 0.6 mm proud of the body where it sits.
    val rings = marks.map(k ->
        val z = height_for(k * 100ml)
        tube(radius_at(z) + 0.6mm, radius_at(z) - wall / 2, 0.6mm).move(z = z))
    hollow + neck + ridge + union(rings)

screw_lid() -> Solid
    val inner_r = major / 2 + clearance
    val outer_r = inner_r + lid_wall
    val skirt = neck_h
    val cap = cylinder(outer_r, skirt + lid_wall) - cylinder(inner_r, skirt)
    val groove = thread(major, pitch, skirt - 1mm, kind = Internal, clearance = clearance)
    // Holes in the top: one in the middle, six around it, twelve outside them.
    val hole = cylinder(hole_d / 2, lid_wall + 2mm).move(z = skirt - 1mm)
    val holes = hole + hole.polar(6, 0.3 * inner_r) + hole.polar(12, 0.62 * inner_r, phase = 15deg)
    val knurl = cube(1.2mm, 1.2mm, skirt).polar(36, outer_r)
    cap - groove - holes + knurl

check wall >= 2 * nozzle else "wall ${wall}, at least two nozzle widths (${2 * nozzle})"
check clearance >= nozzle / 2 else "a thread clearance under ${nozzle / 2} prints fused"

// The base's circular edges are rounded.
part bottle = body().fillet(e -> e.circular and e.center.z < 1mm, 3mm)
part lid = screw_lid().move(x = 3 * radius)

report capacity = capacity
report rings = marks.map(k -> height_for(k * 100ml))
```

### `engine.siml`

```siml
// A single-cylinder four-stroke engine and its inline-four variant: parametric parts, a
// slider-crank driven through 720 degrees, a 0-D Otto cycle with Wiebe heat release, and a sweep of
// the compression ratio.

// -- parameters: the inspector's controls ------------------------------------------------------

param cylinders = 1 in [1, 4]
param bore = 86mm in 60mm..110mm by 0.5mm
param stroke = 86mm in 50mm..110mm by 0.5mm
param rod_length = 143.5mm in 100mm..200mm by 0.5mm
param compression = 10.0 in 8.0..12.0 by 0.1
/// Before firing TDC.
param spark = 20deg in 0deg..45deg by 1deg
/// The Wiebe function's duration.
param burn = 55deg in 30deg..90deg by 1deg
param speed = 3000rpm in 1000rpm..7000rpm by 100rpm

// -- fixed design choices ----------------------------------------------------------------------

val head_dome = 44<cm^3>        // the head casting's chamber; the deck moves to suit
val gasket = 0.9mm              // compressed thickness, bore-sized
val compression_height = 32mm   // wrist pin to crown
val skirt = 30mm                // wrist pin to skirt bottom
val pin_r = 11mm
val rod_beam = 22mm             // the rod's width in the plane it swings in
val min_clearance = 0.8mm       // piston to head at TDC, the least that may be built
val pitch_y = 96mm              // cylinder spacing along the crank

val piston_mass = 320g
val rod_mass = 520g
val steel = 7.85<g/cm^3>
val web_t = 16mm

// Gas and fuel.
val n_poly = 1.3                // polytropic index, compression and expansion
val r_air = 287<J/(kg K)>
val lhv = 44e6<J/kg>
val afr = 14.7
val heat_used = 0.65            // of the fuel's heat: ~0.95 burnt, ~30% of that lost to the walls
val p_intake = 1bar
val p_exhaust = 1.05bar
val t_intake = 330K
val wiebe_a = 5.0
val wiebe_m = 2.0

// -- derived geometry --------------------------------------------------------------------------
// The compression ratio fixes the clearance volume; with the head's chamber and the gasket fixed,
// it fixes how far the crown stops below the deck, and so the block's deck height.

val throw = stroke / 2
val piston_area: cm^2 = pi * bore^2 / 4
val swept: cm^3 = piston_area * stroke

clearance_volume(c: real) -> cm^3 = swept / (c - 1)
deck_gap(c: real) -> mm = (clearance_volume(c) - head_dome - piston_area * gasket) / piston_area
tdc_clearance(c: real) -> mm = deck_gap(c) + gasket
deck_height(c: real) -> mm = throw + rod_length + compression_height + deck_gap(c)

// The dome's depth: a spherical cap of base radius 0.42 bore holding `head_dome`, by Newton.
dome_depth() -> mm
    val a = 0.42 * bore
    var h = 10mm
    for _ in 0..<40
        val v: cm^3 = pi * h * (3 * a^2 + h^2) / 6
        val dv: cm^2 = pi * (a^2 + h^2) / 2
        h -= (v - head_dome) / dv
    h

// The slider-crank: the wrist pin's height above the crank axis, and the crown's drop from TDC.
pin_height(th: deg) -> mm = throw * cos(th) + sqrt(rod_length^2 - throw^2 * sin(th)^2)
drop(th: deg) -> mm = throw + rod_length - pin_height(th)

// d(drop)/dθ: the moment arm of the gas force on the crank, a length per radian.
arm(th: deg) -> mm/rad =
    throw * sin(th) * (1 + throw * cos(th) / sqrt(rod_length^2 - throw^2 * sin(th)^2)) / 1rad

cylinder_volume(c: real, th: deg) -> cm^3 = clearance_volume(c) + piston_area * drop(th)

// Counterweights balance the rotating mass and half the reciprocating mass of each throw. Each
// web carries half; a web's weight is a half-disc of steel of radius R, whose m * centroid is
// rho t (2/3) R^3.
counterweight_radius() -> mm
    val m_rot = rod_mass * 2 / 3
    val m_rec = piston_mass + rod_mass / 3
    val moment = (m_rot + 0.5 * m_rec) * throw / 2
    (3 * moment / (2 * steel * web_t))^(1/3)

// -- parts -------------------------------------------------------------------------------------
// Each is built in its own frame: the piston's origin at the wrist pin, the rod's at its big end
// along +Z, the crank's on its axis along Y, the block and head in the world's frame.

// A cylinder across Y, centred on the origin: a pin, an eye or a journal.
eye(r: mm, width: mm) -> Solid = cylinder(r, width, center = true).turn(X, 90deg)

piston() -> Solid
    val r = bore / 2 - 0.04mm
    val body = cylinder(r, compression_height + skirt).move(z = -skirt)
    // Three ring grooves, each a depth below the crown and a width: top, second, oil.
    val grooves = [(4mm, 1.2mm), (7.5mm, 1.5mm), (11mm, 2.5mm)].map((below, width) ->
        tube(r + 1mm, r - 4mm, width).move(z = compression_height - below - width))
    body.shell(3mm, open = bottom) - union(grooves) - eye(pin_r, bore)

rod() -> Solid
    val beam = cube(rod_beam, 18mm, rod_length)
    val ends = eye(0.32 * stroke, 22mm) + eye(pin_r + 5mm, 22mm).move(z = rod_length)
    val bores = eye(0.24 * stroke, 30mm) + eye(pin_r, 30mm).move(z = rod_length)
    beam + ends - bores

// Inline-four pins: 1 and 4 up, 2 and 3 down (k counts from 0).
pin_phase(n: int, k: int) -> deg = if n == 1 or k == 0 or k == 3 then 0deg else 180deg

// Firing order 1-3-4-2: cylinder k fires this many crank degrees after cylinder 1.
firing_offset(n: int, k: int) -> int = if n == 1 then 0 else [0, 540, 180, 360][k]

crankshaft(n: int) -> Solid
    val cw = counterweight_radius()
    val weight = eye(cw, web_t) * cube(2 * cw, web_t, cw).move(z = -cw)
    val web = cube(50mm, web_t, throw + 25mm).move(z = -10mm) + weight
    val one_throw = eye(0.24 * stroke, 24mm).move(z = throw) + web.move(y = -20mm) + web.move(y = 20mm)
    val throws = (0..<n).map(k -> one_throw.turn(Y, pin_phase(n, k)).move(y = k * pitch_y))
    val journals = (0..n).map(k -> eye(30mm, 24mm).move(y = (k - 0.5) * pitch_y))
    union(throws) + union(journals)

engine_block(n: int, c: real) -> Solid
    val h = deck_height(c)
    val box = cube(bore + 30mm, n * pitch_y, h - 60mm).move(y = (n - 1) * pitch_y / 2, z = 60mm)
    box - union((0..<n).map(k -> cylinder(bore / 2, h).move(y = k * pitch_y, z = 60mm)))

cylinder_head(n: int, c: real) -> Solid
    val z0 = deck_height(c) + gasket
    val depth = dome_depth()
    val a = 0.42 * bore
    // The sphere whose cap of base radius `a` is `depth` deep.
    val sr = (a^2 + depth^2) / (2 * depth)
    val slab = cube(bore + 30mm, n * pitch_y, 90mm).move(y = (n - 1) * pitch_y / 2, z = z0)
    slab - union((0..<n).map(k -> sphere(sr).move(y = k * pitch_y, z = z0 + depth - sr)))

// -- the mechanism -----------------------------------------------------------------------------
// The crank on a driven revolute, a rod on each pin, a piston on each rod sliding in Z.

struct Engine
    mech: Mechanism
    main: Revolute
    head: Body
    rods: [Body]
    pistons: [Body]

engine(n: int, c: real) -> Engine
    val ground = Body("block", engine_block(n, c), 30kg, fixed = true)
    val head = Body("head", cylinder_head(n, c), 12kg, fixed = true)
    val crank = Body("crank", crankshaft(n), 14kg)
    val rods = (0..<n).map(k -> Body("rod${k + 1}", rod(), rod_mass))
    val pistons = (0..<n).map(k -> Body("piston${k + 1}", piston(), piston_mass))
    val main = Revolute(ground.at(origin), crank.at(origin), axis = Y)
    val pins = (0..<n).map(k ->
        val at = vec(0mm, k * pitch_y, throw).turn(Y, pin_phase(n, k))
        Revolute(crank.at(at), rods[k].at(origin), axis = Y))
    val wrists = (0..<n).map(k ->
        Revolute(rods[k].at(vec(0mm, 0mm, rod_length)), pistons[k].at(origin), axis = Y))
    val bores = (0..<n).map(k ->
        Prismatic(ground.at(vec(0mm, k * pitch_y, 0mm)), pistons[k].at(origin), axis = Z))
    val m = Mechanism([ground, head, crank] ++ rods ++ pistons, [main] ++ pins ++ wrists ++ bores,
                      drive = Speed(main.angle, speed))
    Engine(m, main, head, rods, pistons)

// The rod's clearance to the bore's bottom edge at the liner's bottom `z_liner`: the rod's beam is
// `rod_beam` wide and leans by the angle between its pins.
rod_wall_clearance(st: State, e: Engine, k: int, z_liner: mm) -> mm
    val a = st.point(e.rods[k], origin)
    val b = st.position(e.pistons[k])
    val x = a.x + (b.x - a.x) * (z_liner - a.z) / (b.z - a.z)
    bore / 2 - (abs(x) + rod_beam / 2 / cos(st.angle(e.rods[k])))

struct Kinematics
    worst_follow: mm        // the largest |solver - slider-crank| over the run
    head_gap: mm            // the least crown-to-head distance, from the slider-crank
    head_gap_query: mm      // the same, from the solver's min_gap query
    wall_gap: mm            // the least rod-to-bore-edge clearance
    wall_gap_at: deg

kinematics(n: int, c: real) -> Kinematics
    val e = engine(n, c)
    val z_liner = rod_length - skirt + 3mm - throw
    val z_head = deck_height(c) + gasket
    simulate(e.mech, e.main.angle, 0deg..720deg, step = 1deg)
        var worst = 0mm
        var head_gap: mm = inf
        var head_query: mm = inf
        var wall: mm = inf
        var wall_at = 0deg

        on step(st)
            val th = st[e.main.angle]
            for k in 0..<n
                val z = st.position(e.pistons[k]).z
                worst = max(worst, abs(z - pin_height(th + pin_phase(n, k))))
                head_gap = min(head_gap, z_head - (z + compression_height))
                val w = rod_wall_clearance(st, e, k, z_liner)
                if w < wall
                    wall = w
                    wall_at = th
            if th % 360deg == 0deg
                head_query = min(head_query, st.min_gap(e.pistons[0], e.head))

        on contact(st, a, b)
            warn("${a.name} meets ${b.name} at ${st[e.main.angle]}")

        Kinematics(worst, head_gap, head_query, wall, wall_at)

// -- the Otto cycle, single zone ---------------------------------------------------------------
// Crank angle from 0 at the intake TDC; firing TDC at 360. Intake and exhaust are at constant
// pressure; from intake BDC (180) to exhaust BDC (540) the pressure follows
//     dp/dθ = -n p/V dV/dθ + (n - 1)/V dQ/dθ
// with Q the Wiebe function's share of the fuel's heat.

struct Cycle
    p: [bar]                // at 0, 1, …, 719 degrees
    v: [cm^3]
    torque: [N m]           // the gas force's torque on the crank
    work: J
    imep: bar
    peak: bar
    peak_at: deg

burnt(th: deg) -> real
    val start = 360deg - spark
    if th <= start
        0.0
    elif th >= start + burn
        1.0
    else
        1 - exp(-wiebe_a * ((th - start) / burn)^(wiebe_m + 1))

dpdth(c: real, th: deg, p: bar, q_total: J) -> bar/deg
    val v = cylinder_volume(c, th)
    val h = 0.01deg
    val dq: J/deg = q_total * (burnt(th + h) - burnt(th - h)) / (2 * h)
    -n_poly * p / v * piston_area * arm(th) + (n_poly - 1) / v * dq

otto(c: real) -> Cycle
    val air: kg = p_intake * cylinder_volume(c, 180deg) / (r_air * t_intake)
    val q_total: J = air / afr * lhv * heat_used
    var p = fill(720, p_intake)
    var v = fill(720, 0<cm^3>)
    var torque = fill(720, 0<N m>)
    var peak = p_intake
    var peak_at = 0deg
    for i in 0..<720
        val th = i * 1deg
        v[i] = cylinder_volume(c, th)
        if i <= 180
            p[i] = p_intake
        elif i >= 540
            p[i] = p_exhaust
        else
            // Heun's method over the degree, in four quarter-degree steps.
            var q = p[i - 1]
            for sub in 0..<4
                val h = 0.25deg
                val a = th - 1deg + sub * h
                val k1 = dpdth(c, a, q, q_total)
                val k2 = dpdth(c, a + h, q + h * k1, q_total)
                q += h * (k1 + k2) / 2
            p[i] = q
        if p[i] > peak
            peak = p[i]
            peak_at = th
        torque[i] = (p[i] - p_intake) * piston_area * arm(th)
    var work: J = 0
    for i in 0..<720
        val j = (i + 1) % 720
        work += (p[i] + p[j]) / 2 * (v[j] - v[i])
    Cycle(p, v, torque, work, (work / swept) as bar, peak, peak_at)

// -- the inline four ---------------------------------------------------------------------------

// Each cylinder's torque trace shifted by its firing offset, and summed.
summed_torque(cy: Cycle, n: int) -> [N m] =
    (0..<720).map(i -> sum((0..<n).map(k -> cy.torque[(i - firing_offset(n, k)) % 720])))

// The crank's pins and the firing order must agree: cylinder k sits at crank angle
// `θ + pin_phase`, and is at `θ - firing_offset` in its own cycle.
check (0..<4).all(k -> (pin_phase(4, k) + firing_offset(4, k) * 1deg) % 360deg == 0deg)
    else "the inline four's pins and its firing order disagree"

struct Four
    mean: N m
    peak: N m
    least: N m
    power: kW

four(cy: Cycle) -> Four
    val t = summed_torque(cy, 4)
    val mean = sum(t) / 720
    Four(mean, max(t), min(t), mean * speed)

// -- the compression sweep ---------------------------------------------------------------------

struct SweepRow
    compression: real
    deck_gap: mm
    tdc_clearance: mm
    simulated: mm
    imep: bar
    buildable: bool

sweep() -> [SweepRow] =
    (0..8).map(i ->
        val c = 8 + 0.5 * i
        val simulated = kinematics(1, c).head_gap
        SweepRow(c, deck_gap(c), tdc_clearance(c), simulated, otto(c).imep, simulated >= min_clearance))

// -- the document ------------------------------------------------------------------------------

check tdc_clearance(compression) >= min_clearance
    else "piston ${tdc_clearance(compression)} from the head at TDC, minimum ${min_clearance}"

part assembly = engine(cylinders, compression).mech

report swept = swept
report clearance_volume = clearance_volume(compression)
report deck_height = deck_height(compression)
report dome_depth = dome_depth()
report counterweight = counterweight_radius()

study kinematics = kinematics(cylinders, compression)
study cycle = otto(compression)
study inline_four = four(otto(compression))
study sweep = sweep()
```

What the inspector shows for this file: eight controls; five live numbers; four studies, of which
`cycle` plots `p`, `v` and `torque` against their index, and `sweep` is a table of nine rows with the
two unbuildable compressions marked by their `buildable` column. The `check` on the TDC clearance turns
the probe's printed "no: piston meets head" into a warning at the `compression` slider's line as soon as
the slider passes 11.

## 13. Open questions, and a staged plan

Each question has a recommendation; none blocks writing stage 1.

1. **Definitions: `name(params) -> T` (sysl, slate) or `def` (nex)?** *Recommend sysl's form*, as
   written here, so the three languages of the org read alike. The cost is losing source compatibility
   with nex scripts, which no user has yet.
2. **Indexing from 0 or from 1?** nex counts from 1 (Fortran); sysl and slate from 0. *Recommend 0*,
   for the same reason, and because Tekton's own code, which hosts the scripts, counts from 0.
3. **A fork of nex's front end, or a shared library both languages configure?** *Recommend the
   fork*, with units extracted to `sysl-lang/units` first. nex's elaborator carries rank-N arrays,
   `inout`, parallelism and FFI through most of its files; a configurable front end would be both
   languages' complexity in one place.
4. **What may a drag rewrite?** §4 never changes a `param` and appends `+ d` instead. *Recommend
   keeping that*, and adding later an explicit handle a script declares (`handle radius on
   bottle.faces(top)`), which binds a viewport handle to a parameter on purpose rather than by
   inference.
5. **Constrained sketches.** A sketch solved from constraints (coincident, tangent, distance) is what
   CAD users expect for profiles, and needs a native geometric constraint solver. *Recommend profiles
   as values in stage 1 and a constraint sketch as a later value type*, its solver native in Tekton
   like the others.
6. **Fillet and chamfer on a mesh kernel.** Manifold has no general edge fillet; Tekton's rounded
   boxes are hulls built per primitive. *Recommend*: in stage 1, `fillet` and `chamfer` only on a
   primitive's own edges (what Tekton already does, reached by a selector); general fillets are a
   kernel question, not a language one, and are reported as such when a script asks for one.
7. **Does a document drawn in Tekton's GUI become a siml file?** *Recommend one-way export* (GUI to
   siml, writing `part`s with literal transforms, which the drag rules then edit) and no merging
   back. A script is a document kind of its own.
8. **Studies on every tick, or on demand?** *Recommend on demand*, with `live study` as an opt-in for
   one that fits the 50 ms budget, since the deadline (§9) already keeps a slow one from blocking the
   viewport.
9. **Acausal modelling.** *Recommend no*, until a concrete part needs it (§7).
10. **The name and the file extension.** `siml` and `.siml`, unless the user prefers otherwise.

**Staged plan.**

- **Stage 0, the shared pieces.** `sysl-lang/units` extracted from nex, with nex consuming it;
  source edits and the decimal writer added to `parsing`. Each is released before siml uses it.
- **Stage 1, a first usable siml.** Lexer, parser and elaborator for the core: `param`, `val`,
  functions, structs, enums, `match`, lists, units, `check`, `part`, `report`. The closure
  interpreter. Solids as values over the operations Tekton already has (primitives, CSG, transforms,
  clearances, rounded primitive edges), plus `extrude`, `revolve` and `loft` of profiles. Diagnostics
  at spans in an editor pane. The inspector built from `param`s. Click a face, go to its line.
  *Exit*: `shaker.siml` without the threads runs in Tekton, matches the probe's 380 ml and ring
  heights, and a slider tick is under 50 ms.
- **Stage 2, live editing.** The memo table and its keys, cancellation and deadlines, selection
  highlighting both ways, and the drag rules of §4. Selectors over faces and edges; `shell`; threads.
  *Exit*: the whole of `shaker.siml`; dragging the lid writes `3 * radius + 12mm`; a `radius` tick
  takes the lid's mesh from the cache, shown by a test that counts kernel calls.
- **Stage 3, simulation.** Bodies, frames, joints, ports, drivers, `simulate` with `on step` and
  `on contact`, the queries, `study` with tables and plots, and the scrubber. *Exit*: `engine.siml`
  reproduces `REPORT.md`'s numbers (1.981 mm at TDC, rod clearance 8.21 mm at 65°, IMEP 11.5 bar, 183
  N·m mean and 57.5 kW for the four), each study under 1 s.
- **Stage 4, printing-aware design.** Design rules from the printer profile, mapped to spans;
  constraint sketches; the declared handles of question 4.

Every stage's assertions go into siml's test suite as they are made, the probe's numbers first among
them.
