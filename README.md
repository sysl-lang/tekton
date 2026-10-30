# Tekton

A 3D modeller with an uncomplicated interface, written in [sysl](https://github.com/sysl-lang/sysl).
*Tekton* is Greek for builder. Site: [tekton3d.org](https://tekton3d.org).

So far: put shapes on a ground plane, orbit around them, and select one by clicking. [DESIGN.md](DESIGN.md)
says what it will be.

## Running it

It needs Manifold (`brew install manifold`) and a Mac: the shaders are Metal Shading Language. From a shell
in the desktop session (`launchctl managername` says `Aqua`), with `quaternion` and `sdl3-shapes` checked out
beside this directory:

```
sysl build . --lib ../quaternion --lib ../sdl3-shapes && ./tekton
```

**The two `--lib` flags are for now.** `sysl-lang/quaternion` has no tag to fetch, and `sdl3-shapes` holds the
fix that makes a pipeline's `blend` take effect. Once both are released the command is `sysl build .`.

- **Drag** on empty space or on a shape to orbit, **right- or middle-drag** to pan, **wheel** to zoom.
- **Click** a shape to select it, or empty space to clear the selection.
- `./tekton --demo` opens with one of each shape.
- `./tekton --demo --shot frame.png --size 1400x900` writes a frame to a PNG and closes its window.

## Tests

```
sysl test . --lib ../quaternion --lib ../sdl3-shapes
```

The document, camera, picking and pointer tests need neither a window nor a GPU. The renderer's three draw real
pixels on a device with no window, and outside the desktop session they say they did not run and pass.

## The modules

Under `sh/sysl/tekton/`, and `main.sysl` only wires them:

| module | what | imports |
|---|---|---|
| `space` | vectors, rays, boxes, a matrix | nothing |
| `document` | the tree of operations, its commands, and the kernel's meshes cached per node | Manifold |
| `camera` | the orbit camera, its clamps, its rays and its matrix | quaternion |
| `picking` | which shape a ray meets first | |
| `renderer` | draws a document into any targets it is handed | SDL_GPU |
| `interface` | the palette and viewport, and the pointer | syslUI, syslui-sdl |
