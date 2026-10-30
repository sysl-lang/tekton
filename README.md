# Tekton

A 3D modeller with an uncomplicated interface, written in [sysl](https://github.com/sysl-lang/sysl).
*Tekton* is Greek for builder. Site: [tekton3d.org](https://tekton3d.org).

So far: put shapes on a ground plane, orbit around them, select them, move, turn and resize them with handles or
typed numbers, make them solids or holes, group them to join the solids and cut the holes, and undo any of it.
[DESIGN.md](DESIGN.md) says what it will be.

## Running it

It needs Manifold (`brew install manifold`) and a Mac: the shaders are Metal Shading Language. From a shell
in the desktop session (`launchctl managername` says `Aqua`):

```
sysl build . && ./tekton
```

- **Drag** on empty space to orbit, **right- or middle-drag** to pan, **wheel** to zoom.
- **Click** a shape to select it, or empty space to clear the selection.
- **Drag a shape** to slide it along the ground.
- On the selected shape: the **arrow** above it raises it; the **dots on the corners of its base** resize it and
  the **dot on top** changes its height; the three **curved arrows** turn it about the red, green and blue axes.
- Drags snap to a 1 mm grid and turns to 15 degrees. **Shift** lets go of the snap, and **Option** keeps a resize
  in proportion. No side goes below 1 mm.
- The panel shows the selection's position, size and rotation. Type a number and press Return, or click away;
  Escape puts the old one back, and a value it cannot take is marked and leaves the shape as it was.
- **Shift-click** adds a shape to the selection or takes it away. Several selected show no handles and no numbers:
  dragging any of them moves them all.
- **Solid** and **Hole** in the panel say which the selection is. A hole is drawn translucent, and faintly through
  whatever it stands inside, until it is grouped.
- **Group** (Cmd+G) makes one thing of what is selected, standing where it stood: the solids joined, the holes cut
  out. **Ungroup** (Cmd+Shift+G) gives the parts back, each where it stands now, and groups nest. A group with no
  solid in it, or holes that would leave nothing, is refused, and the panel says why. A part that a stretched group
  sheared stays in a group of its own, since a shape's transform cannot say a shear.
- **Delete** or Backspace removes the selection. **Undo** (Cmd+Z) and **Redo** (Cmd+Shift+Z) reach every edit, with
  one drag or one typed number a step, and put the selection back with it. History holds 200 steps, or fewer once
  the meshes only it holds pass 256 MB. While a field has the keyboard, these keys belong to the field.
- `./tekton --demo` opens with one of each shape. `--select 3` chooses what is selected (`--select 3,4` for several)
  and `--view 0.3,0.5` sets the camera's turn and tilt in radians. `./tekton --holes` opens with a part with a hole
  cut in it, beside a hole not yet grouped.
- `./tekton --demo --shot frame.png --size 1400x900` writes a frame to a PNG and closes its window.

## Tests

```
sysl test .
```

The document, camera, picking, handle and pointer tests need neither a window nor a GPU. The renderer's draw real
pixels on a device with no window, and outside the desktop session they say they did not run and pass.

## The modules

Under `sh/sysl/tekton/`, and `main.sysl` only wires them:

| module | what | imports |
|---|---|---|
| `space` | vectors, rays, boxes, a matrix | nothing |
| `document` | the tree of operations, its commands and undo, and the kernel's meshes cached per node, each in its node's own space. Nodes are replaced, not edited, so old roots are the history | Manifold |
| `camera` | the orbit camera, its clamps, its rays and its matrix | quaternion |
| `picking` | which shape a ray meets first | |
| `handles` | where the selection's handles land on the screen, which one a point is on, and what a drag does | quaternion |
| `renderer` | draws a document and a handle layout into any targets it is handed | SDL_GPU |
| `interface` | the palette, viewport, selection panel and number fields, the pointer and the shortcuts | syslUI, syslui-sdl |
