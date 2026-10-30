# Tekton design

## What it is

A modeller for building solid objects, and a stressor for sysl and syslUI. Every language gap it finds
is reported rather than worked around.

## The interface

A model is made from a few ideas:

- **Shapes.** Pick a box, cylinder, sphere, cone or torus and drop it on the ground plane.
- **Move, rotate, resize.** Drag handles on the selected shape, snapped to a grid. Numbers can be typed.
- **Solid or hole.** Every shape is one or the other. Grouping shapes joins the solids and cuts the
  holes out of them.
- **Group and ungroup.** A group stays editable, because it remembers its parts.

The window is one viewport, a short shape palette and a small panel for the selection. Undo reaches
everything. There are no modes: no edit mode, no modifier stack, no vertex editing.

## The document

A document is a tree of operations, not a mesh: shapes and groups, each with a transform and a
solid-or-hole flag. Meshes are derived from it by the kernel and cached per node, so an edit recomputes
only the path from that node to the root. This same tree is what a script will produce.

## The layers

| layer | what |
|---|---|
| kernel | `sysl-lang/manifold`: watertight booleans and primitives |
| viewport | SDL_GPU through `sysl-lang/sdl3`, with a depth buffer |
| interface | `sysl-lang/syslui` on `sysl-lang/syslui-sdl`, drawn over the viewport |
| files | a text document format; STL and 3MF export |
| scripting | interpreted nex, through its embedding API |

SDL's `SDL_CreateGPURenderer` lets one GPU device drive both the 2D renderer syslui-sdl already uses
and the 3D pass, so the interface and the viewport share a window.

## Driving it without a person

Everything runs without a window, so a test or an agent can use the application the way a person does:

- **The document is a library.** Every edit is a command with no interface attached, so model
  behaviour is tested directly, and undo is tested against every command.
- **A running application can be controlled live, the way a browser is through its developer tools.**
  Started with `--control <socket>`, in a window or offscreen, it serves HTTP on a private Unix socket, so
  `curl --unix-socket <socket> http://tekton/snapshot` is a complete client. Its commands are: `snapshot` (the controls on screen, each with a reference), `click`, `drag`, `key`, `type`,
  `shot <file>`, `wait` and `state`, which prints the document, selection and camera as text. Input
  feeds the same event path a real window does, so a click that misses a handle misses here too. A
  script is a file of the same commands. Without the flag, nothing listens.
  **A person can watch it happen.** Control runs in a real window by default, and offscreen only when
  asked. Controlled input draws a visible pointer and press marker, and a drag moves across the screen
  at a watchable pace rather than jumping to its end.
- **Screenshots are real frames**: the same GPU pass and interface, read back into a PNG. Pixel
  assertions run against a fixed camera and size.

Most of this belongs below Tekton, so every application gets it. sdl3 owns the socket, input
by coordinates through `SDL_PushEvent` (the real event path), and reading a frame back into a PNG, so any
SDL program can be controlled. syslUI adds what SDL cannot know: named controls and the snapshot of what
is on screen. syslui-sdl joins the two. Tekton adds only its own `state` report and its viewport.

**Building stays fast by keeping layers apart.** The document, kernel glue and renderer are modules with
their own tests, so most changes never need the whole application linked or a window opened.

## Stages

1. SDL_GPU in the sdl3 binding, proved by rendering offscreen and reading the pixels back.
2. A viewport in syslUI and syslui-sdl: a view whose rectangle the application draws 3D into, plus the
   offscreen driver.
3. Shapes, orbit camera, picking, and handles.
4. Solid or hole, groups, undo.
5. Saving, STL and 3MF export.
6. nex scripting.
