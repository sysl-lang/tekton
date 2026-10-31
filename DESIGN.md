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

The window is a menu bar across the top and, below it, one viewport, a short shape palette and a small
panel for the selection. Undo reaches everything. There are no modes: no edit mode, no modifier stack, no
vertex editing.

The bar holds File, Edit and View. File has New, Open, Save, Save As, Export STL… and Quit, with Cmd+N, Cmd+O,
Cmd+S and Cmd+Shift+S. Every other row is a command the palette, the panel or a key already reaches. Edit has Undo, Redo, Delete, Group, Ungroup, Solid and Hole. View has Frame All. A
row is dimmed where its command would do nothing now, and its shortcut is shown only where that key works.
While a menu is open the viewport takes no input at all, so a press, a drag or the wheel over a menu or
beside it never reaches the model or the camera.

## The document

A document is a tree of operations, not a mesh: shapes and groups, each with a transform and a
solid-or-hole flag. Meshes are derived from it by the kernel and cached per node, so an edit recomputes
only the path from that node to the root. This same tree is what a script will produce.

## Files

A document is saved as plain text in `sh.sysl.tekton.files`: HOCON whose first line is `tekton = 1`, the format
version, read by itself before anything else, so a newer format is told "too new" whatever else it holds. It
holds the tree, `next` (the id the next node gets) and `last` (what the next shape is placed beside). Meshes are
derived, and the selection and the history belong to a session. Every number is written as the shortest text
that reads back as the same value, so a saved document comes back bit for bit.

A bad file is refused with its line and the problem, and whatever is open stays as it was. That covers more than
syntax: an id used twice, a transform no edit could make, groups nested past 128, a group the kernel cannot
join. A document that would be refused is not written either.

**Saving is atomic**: the text goes to a name beside the target and is renamed onto it, so a failure leaves the
old file whole, and the panel says why.

**Unsaved means different from what the file holds**, compared by value, so undoing back to what was saved is
clean. The window's title is the file's name, or `Untitled` before it has one, then `— Tekton`, with `• ` in
front while anything is unsaved (`window_title`); the panel shows the file's name under its heading. New, Open,
Quit and closing the window ask first, in the system's own message box attached to the window: Save, Don't Save
or Cancel, with Return as Save and Escape as Cancel. What an answer means is a pure function (`unsaved_answer`):
a box closed without a choice is Cancel, and a box that could not be shown is Cancel too, with the reason in the
panel. The box holds the whole application until it is answered, the control socket included. Closing the window
sends two events, and only the first in a frame asks. Save with no file yet goes through Save As, and if that is
cancelled or fails, what was going to happen does not.

Open has no undo to offer: it clears the history and frames the camera on what it loaded. `tekton <file>` opens a
file at startup; a bad one is reported and it starts empty, with no file, so Save cannot overwrite it.

File dialogs are SDL's. What an answer means is a pure function (`open_answer`, `save_answer`), and while a dialog
is up nothing behind it takes input, because what it answers was asked about this document. The dialogs and the
question are given the window `syslui-sdl` opened (`window()`), so they belong to it.

## Export

**File > Export STL…** writes what a slicer is sent, in `sh.sysl.tekton.export`: the document's one final solid
(`Document.final_solid`), every part joined and every hole cut as if the whole document were one group, so a hole
standing on the ground cuts whatever it overlaps. It is binary STL in millimetres: an 80-byte header that says
Tekton (and so never begins with `solid`, which marks a text STL), the triangle count, and each triangle's unit
normal and three corners, counter-clockwise from outside, as the kernel's own little-endian `f32`s.

A document with nothing to send says why in the panel before any dialog, and nothing is written: one a file
would refuse (a group with no solid part, one the kernel cannot join), one with no solid at all, or one whose
holes cut everything away. The file is written atomically like a save, so a failure leaves the old one whole. The
dialog filters for `.stl`, and the answer gets `.stl` added unless it already ends that way in any case
(`export_answer`), since a slicer goes by the extension. **An export is not a save**: the document keeps its file
and its unsaved mark.

3MF's three parts are made and tested (`content_types_part`, `relationships_part`, `model_part`): the model in
millimetres, each distinct position once with indexed triangles, any name XML-escaped. **There is no Export 3MF
yet**, because 3MF is a zip and no package in the org writes one: `sysl-lang/miniz` binds deflate and CRC-32 with
miniz's archive API compiled out.

## The layers

| layer | what |
|---|---|
| kernel | `sysl-lang/manifold`: watertight booleans and primitives |
| viewport | SDL_GPU through `sysl-lang/sdl3`, with a depth buffer |
| interface | `sysl-lang/syslui` on `sysl-lang/syslui-sdl`, drawn over the viewport |
| files | a text document format through `sysl-lang/hocon`; binary STL export |
| scripting | interpreted nex, through its embedding API |

SDL's `SDL_CreateGPURenderer` lets one GPU device drive both the 2D renderer syslui-sdl already uses
and the 3D pass, so the interface and the viewport share a window.

## Driving it without a person

Everything runs without a window, so a test or an agent can use the application the way a person does:

- **The document is a library.** Every edit is a command with no interface attached, so model
  behaviour is tested directly, and undo is tested against every command.
- **A running application can be controlled live, the way a browser is through its developer tools.**
  Started with `--control <socket>`, it serves HTTP on a private Unix socket through `syslui-sdl`, so
  `curl --unix-socket <socket> http://tekton/snapshot` is a complete client. The routes are `GET /snapshot`
  (the controls on screen, each with a reference; the palette's are `palette.box` and so on),
  `POST /click?ref=<name>`, and sdl3's `/click`, `/drag`, `/key`, `/type`, `/wait`, `/state` and `/shot`.
  Input feeds the same event path a real window does, so a click that misses a handle misses here too, and
  `/shot` is the real frame, the 3D viewport included. Without the flag, nothing listens.
  **A person can watch it happen.** Control runs in the real window, and the controlled pointer is drawn
  over the frame with a mark while a button is held; a drag moves at a watchable pace.
  **Tekton's own two routes**, added through `syslui-sdl`'s `routes` (asked before the built-ins; everything
  else passes on to them):
  - `GET /tekton/state` answers one line of JSON: `items` (the document's tree: each node's `id`, `kind`,
    `hole`, `position`, `size` in millimetres along its own axes, `rotation` as `[w,x,y,z]`, and for a group
    its `children`), `selection` (ids, in the order chosen), `camera` (`target`, `distance`, `yaw`, `tilt`,
    `fov`), `file` (the path, or `null`) and `dirty`. A number JSON cannot spell is `null`.
  - `POST /tekton/quit` quits at once, asking nothing about unsaved changes, so a session nobody can answer
    can be ended; the answer is sent, and the program exits on the next frame. A `GET` of it is refused.
    The window's own close still asks.

  **`--hidden`**, meant with `--control`, creates the window hidden: it paints every frame as usual,
  `/shot` reads it back, and nothing appears on the screen. It still needs a desktop session (Aqua) to
  open at all; there is no headless mode below that.
- **Screenshots are real frames**: the same GPU pass and interface, read back into a PNG. Pixel
  assertions run against a fixed camera and size. `--menu <title>` takes one with that menu open.

Most of this belongs below Tekton, so every application gets it. sdl3 owns the socket, input
by coordinates through `SDL_PushEvent` (the real event path), and reading a frame back into a PNG, so any
SDL program can be controlled. syslUI adds what SDL cannot know: named controls and the snapshot of what
is on screen. syslui-sdl joins the two. Tekton adds only its own `state` and `quit` routes and its viewport.

**Building stays fast by keeping layers apart.** The document, kernel glue and renderer are modules with
their own tests, so most changes never need the whole application linked or a window opened.

## Stages

1. SDL_GPU in the sdl3 binding, proved by rendering offscreen and reading the pixels back.
2. A viewport in syslUI and syslui-sdl: a view whose rectangle the application draws 3D into, plus the
   offscreen driver.
3. Shapes, orbit camera, picking, and handles.
4. Solid or hole, groups, undo.
5. Saving and opening, then STL and 3MF export.
6. Printing: the printer's bed in the viewport, control over facets on round shapes, measuring, align, mirror
   and array, hole clearances, chamfers, text.
   - Mirror and align, in an Arrange menu: Mirror X/Y/Z reflects the selection along a ground axis through the
     middle of its box, stored as a node's reflection along its own nearest axis (`mirror` in a file) so a
     transform's scale stays positive and exported triangles face outward; Align lines up the low side, middle
     or high side of two or more boxes on one axis; Drop to Bed (Cmd+B) lowers or raises the selection onto z = 0.
     Each is one undo step.
7. nex scripting.
