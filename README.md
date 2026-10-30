# Tekton

A 3D modeller with an uncomplicated interface, written in [sysl](https://github.com/sysl-lang/sysl).
*Tekton* is Greek for builder. Site: [tekton3d.org](https://tekton3d.org).

So far: put shapes on a ground plane, orbit around them, select one by clicking, and move, turn and resize it
with handles or typed numbers. [DESIGN.md](DESIGN.md) says what it will be.

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
- `./tekton --demo` opens with one of each shape. `--select 3` chooses which is selected and `--view 0.3,0.5` sets
  the camera's turn and tilt in radians.
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
| `document` | the tree of operations, its commands, and the kernel's meshes cached per node, each in its node's own space | Manifold |
| `camera` | the orbit camera, its clamps, its rays and its matrix | quaternion |
| `picking` | which shape a ray meets first | |
| `handles` | where the selection's handles land on the screen, which one a point is on, and what a drag does | quaternion |
| `renderer` | draws a document and a handle layout into any targets it is handed | SDL_GPU |
| `interface` | the palette, viewport and number fields, and the pointer | syslUI, syslui-sdl |
