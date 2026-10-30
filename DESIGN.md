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

## Stages

1. SDL_GPU in the sdl3 binding, proved by rendering offscreen and reading the pixels back.
2. A viewport in syslUI and syslui-sdl: a view whose rectangle the application draws 3D into.
3. Shapes, orbit camera, picking, and handles.
4. Solid or hole, groups, undo.
5. Saving, STL and 3MF export.
6. nex scripting.
