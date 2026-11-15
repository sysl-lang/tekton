# nex as Tekton's scripting language — probe report

Probe scripts: `shaker.nex`, `engine.nex` (modes `parts | kin1 | kin4 | cycle | four | sweep | all`), against
the stub API `tekton/tekton.nex` and its host-shaped twin `host/tekton_host.nex`. One reduction per finding in
`gaps/g01`–`g13`. Run on nex 0.0.3's interpreter (`nex run --interp`).

## Verdict

**nex fits, with 3 bugs and 4 missing features, plus 7 style frictions.** For setting up, driving and querying a
simulation it is very good: units of measure kept every quantity right through the Otto model (bar, cm³, J/deg,
N·m, turn/s → kW) with no hand conversions, and the diagnostics are the best part of the language. Describing
parts is where it falls short: no operators on user types (no `a + b` for CSG), no array concatenation, and an
enum can hold itself only through an array. A dedicated language is not needed on this evidence; operators on
user types and array concatenation would close most of the gap.

## Results (they run for real)

86×86 mm bore × stroke, rod 143.5 mm, compression ratio 10.

- **Piston follow:** the joint solver and the slider-crank formula agree to 1e-13 mm over all 720 steps.
- **Clearances:** piston to head at TDC 1.981 mm (formula and `min_gap` agree); rod to bore edge at least
  8.21 mm, at 65°.
- **Single cylinder:** peak 51.6 bar at 16° after TDC, indicated work 575 J, **IMEP 11.5 bar** (textbook 8–12),
  mean torque 45.8 N·m, which equals W/4π and so checks the integration.
- **Inline four (1-3-4-2):** summed torque mean 183 N·m, peak 611, least −115 N·m; 57.5 kW indicated at
  3000 rpm.
- **Sweep, CR 8–12:** IMEP 11.07 → 11.87 bar; TDC clearance 4.71 → 0.24 mm. CR 8–11 can be built; 11.5 and 12
  cannot (clearance under the 0.8 mm minimum).
- **Shaker:** 380 ml, with 100 ml rings at 33.3, 75.0 and 113.2 mm.

| Run (load + elaborate + run) | Time |
|---|---|
| shaker | 12 ms |
| engine parts, 1 and 4 cylinders | 10 ms |
| Otto cycle | 30 ms |
| 720-step 4-cylinder kinematics | 330 ms |
| CR sweep | 550 ms |
| everything | 0.90 s |

Parts are well under the 50 ms live-slider budget; the simulation runs are under 1 s, and most of their time is
the stub's solver, which would be native in Tekton.

## Bugs (the spec or the CLI's own behaviour says otherwise)

1. **The interpreter panics on a call to a host function** (`gaps/g07g_host_call.nex`):
   `@host extern def area(w: real<mm>): real<mm^2>` and `def main() = print(area(2mm))` →
   `panic: unwrap of a None value (nex/sh/sysl/nex/embed.sysl:659:42)`. Without the call, the proper "gives none
   of that name" diagnostic appears.
2. **Overloading is refused.** Spec §6.4: "Several `def`s may share a name if their parameter types differ";
   `def kind(v: real)` beside `def kind(v: string)` → `` redeclaration of `kind` ``. DESIGN.md:2198 says "no
   overloading by type", so one of the two documents is wrong (`gaps/g10`).
3. **`print` takes one argument.** Spec §12.11 gives `print(x, …)` and §12.8's example uses `print(k, x)`; the
   implementation answers `` `print` takes one value; interpolate several `` (`elab_call.sysl:751`, `gaps/g04`).

## Missing features

1. **Operators on user types** (`gaps/g02`): `` `+` needs numbers; found V and V ``, so CSG is `union(a, b)`,
   `cut_all(a, [...])`, `intersect`. The largest gap for describing parts.
2. **Array concatenation** (`gaps/g03`, `g06`): `[parts, [x]]` makes a rank-2 array; building the four-cylinder
   bodies and joints took `flatMap(0..n, k -> if k == 0 then [...] else [...])`.
3. **A struct generic over a unit** (`gaps/g05`): `struct Param[U: Unit]` →
   `` a unit parameter (`U: Unit`) is not elaborated yet ``.
4. **Metadata on a parameter, and a unit as a value:** nowhere to put a slider's range on `var height = 150mm`;
   an array of parameters holds one unit only, and a unit cannot be named at run time, so `Slider` strips the
   number and spells its unit as text. Natural: `@range(80mm, 220mm) var height = 150mm`, read by the host.

## Style frictions

- An enum holds itself only through an array: `Moved(s: [Solid], …)` with `ss[1]` everywhere (`g01`); the
  diagnostic says so clearly.
- `unit rpm = turn / 60 s` is refused ("the only number in a unit expression is `1`"); needs `unit minute = 60 s`.
- `unit` is a keyword, so no field called `unit`.
- `floor` returns a real and a range refuses one: `to_integer(floor(x))`.
- A loop variable `step` hid the function `step`; the message (`this is int64 and cannot be called`) does not
  mention the shadowing.
- An angle inside a rate silently becomes radians: `2bar / 1deg` prints `114.59 bar`; declaring
  `real<bar/deg>` keeps it. Correct but surprising.
- A trap printed by `nex run` has no location (`trap: assertion failed: loft: one height per section`); the
  embedding API does return the stack.

## Embedding, speed, diagnostics

- **Hosting:** `Solid` and `State` become `@host struct`s and the constructors `@host extern def`s. A host
  function may not have defaults or take a function (clear refusals, `g08`), so the API is two layers: host
  externs plus a nex prelude of `def` wrappers (`opts.prelude`). Scripts never look inside a `Solid`, so they do
  not change.
- **Edge predicates** stay in nex: the host returns `[Edge]` as data, nex filters, ids go back.
- **Per-step simulation** is script-driven: `st = step(st, 1deg)` in a loop, then `position_of` / `min_gap`. The
  host can call a named script function each step (`script.call`), but a script cannot hand the host a closure.
- **Parameters:** the host sets module-level `var`s through `set`, in their declared units; `budget` and
  `cancel()` let a slider drag abandon a stale run.
- **Diagnostics** for typical mistakes are excellent (`g12`): they name the parameter and both units ("argument
  `z` must be real<mm>, but this is real64<deg>: angle and length are different dimensions"), suggest `30<deg>`
  for a bare angle, and report `` `tekton.cylinder` needs its argument `h` `` and
  `` `tekton.shell` has no parameter `opne` ``. All seven mistakes in one file were reported in one pass. No "did
  you mean" suggestions.
