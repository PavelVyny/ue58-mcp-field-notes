# EditorAppToolset

Part of the [register](../README.md#the-register). The markers are explained in [How to read an entry](../README.md#how-to-read-an-entry).

### `CaptureViewport` drops a `captureTransform` written with the wrong keys

**Kind:** defect · **Hit on:** 5.8.0, re-tested on 5.8.3 · **Workaround:** yes

> **Correction (re-tested).** An earlier version of this note said the tool ignores
> `captureTransform` altogether. That was wrong. The pose is applied, but only when it is written
> as `{location: {x, y, z}, rotation: {pitch, yaw, roll}, scale: {x, y, z}}`, the same shape the
> validation error of `SetCameraTransform` prints. Written as
> `{translation, rotation: {x, y, z, w}, scale3D}` it is silently dropped, and the capture comes
> from the world origin. The `cameraLocation` field of the response tells you which happened: it
> reads `0, 0, 0` when the pose was dropped and your coordinates when it was used. The original
> story is kept below because the trap it describes is real.

This one took three rounds to understand, so here is the whole arc.

First it looked like a stale frame: captures lagged the real viewport by several edits. The
obvious fix was to wake the viewport up - `SetCameraTransform` to the same pose, then capture.
That seemed to help, so it went in the notes as the rule.

It did not help. Next came `captureTransform`, the argument that renders from a given pose
without moving the viewport - clean, no nudging. That went in the notes too.

Then I ran three captures with three different `translation` values and got three identical
images, of a part of the level neither I nor the argument had asked for.

The conclusion back then was that the tool returns its own fixed view. The real cause was the key
names: `translation` is not a field the tool reads, so every one of those captures came from the
same default pose.

**Workaround.** Use `location` / `rotation` (as a rotator) / `scale`, and check `cameraLocation`
in the response before trusting the image.

### Optional arguments that are not optional

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

`CaptureViewport` marks `captureTransform` and `annotations` as `TOptional`. Omit either and
the call fails with `input param X needs a default value`. Both are mandatory in practice.
Passing them explicitly as `null` is accepted, though (re-tested on 5.8.3), and that is the
useful form: see the next note about capturing through a camera.

Cause (source-checked, 5.8.3): the schema omits `TOptional` parameters from `required`, but the
invoker demands a `default` for every top-level parameter you leave out
(`JsonSchemaGenerator.cpp`, `ObjectFunctionToolCall.cpp`). An explicit `null` passes that check.
PCG `AddNode`/`UpdateNode` behave the same way for `nodeTitle`/`nodeComment`, whose C++ default is
an empty string (observed; not traced in source).

`StartPIE` wants its `options` block as a parameter. An earlier version of this note said
`playMode` inside it is required even when `bSimulate` already says what you mean. In the 5.8.3
source the fields of that block are declared optional with defaults, and the server does not
check nested fields: in a `CaptureViewport` re-test an `annotations` block with fields left out
was accepted, and the missing ones took the struct's defaults, not zeros. `StartPIE` itself was
not re-tested.

Disabled annotations no longer need a block of zeros plus `classFilter: {"refPath": ""}`:
`annotations: null` works (re-tested on 5.8.3).

**Workaround.** Pass every top-level parameter, with `null` for a `TOptional` you do not need.

### Capturing through a sequence camera, and the camera bodies that block it

**Kind:** note · **Hit on:** 5.8.3 · **Workaround:** yes

To see what a Sequencer camera actually frames, lock the viewport to the camera cuts
(`set_camera_lock(true)`), move the playhead, and call `CaptureViewport` with
`captureTransform: null` and `annotations: null`. The capture then comes from the cut's camera
with that camera's field of view (a 24 mm lens on a 16:9 Digital Film back reports 52.7°, not the
viewport's 90°).

The catch: every cine camera in the editor world draws its body mesh
(`CameraProxyMeshComponent_0`). With several cameras placed around one subject, another camera
sitting on the line of sight fills the frame, and depth of field turns it into a dark, soft blur.
A `trace_world` from the camera to the subject comes back clean, because the body has no
collision, so the symptom looks like an empty or unlit scene.

**Workaround.** For previews, set `bVisible: false` on `CameraProxyMeshComponent_0` of each
spawned camera with `ObjectTools.set_properties`. Neither renders nor games ever draw those
bodies. Also move the playhead between captures; the first capture after a change can still be
the previous frame.

### `CaptureViewport` returns ~2.8 MB of base64 inline

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

In my client (Claude Code) a response this large spills into a `tool-results` file, and the
client has to go read it. That is the client, not the tool: the tool returns the image inline.
Worth knowing before you plan a loop around it.

The structure is `returnValue.image.data` (`FViewportCapture.Image`, an `FToolsetImage` with
`Data`), not `returnValue.data`. Sibling tools differ here:
the Slate inspector's screenshot puts its payload directly at `returnValue.data`. Same idea,
different shape, no warning.

**Workaround.** Parse the file, decode `returnValue.image.data`, write a `.png`, read that.

### `GetCameraTransform` only tells the truth while the camera is locked

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

Without `set_camera_lock(true)` it reports the free viewport, not the sequence camera - and
because the viewport eases into position, two reads a second apart give different numbers.
The symptom reads as "my keys are in the wrong place", which sends you to fix the keys.

`close_sequence` and `open_sequence` both drop the lock.

**Workaround.** To verify keys by pose: lock → `set_playhead_frame` → `force_evaluate` →
`GetCameraTransform`.

### There is no console-command tool

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

`EditorAppToolset` can search console variables with `SearchCVars`, which also returns each
variable's current value. It cannot set one, and there is no `ExecuteConsoleCommand` anywhere in
the surface. For a system built to automate the editor,
the absence is louder than most bugs on this page.

**Workaround.** Type into the editor's status-bar command box through the Slate inspector:
`Type {ref, text, submit: true}`. Find the ref with `Observe("")` then `Snapshot` on the status
bar menu - it is the textbox next to "Cmd". It works while PIE is running, too.

With PIE maximized the status-bar box is not in the tree. Use the in-game console instead:
`PressKey {key: "Tilde"}`, `Snapshot`, then `Type` into the focused textbox. The ref is
single-use: after Enter the console closes, and the next open has a new ref (typing into the old
one returns false). Check that a setting took with `SearchCVars`.

### Every `ProfileGPU` leaves a GPU Visualizer window open, and the next profile pays for it

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

The window is never closed. They stack up, they redraw every frame, and the editor starts
crawling - which reads as "profiling made my editor slow" rather than "I have six windows
open". Worse, an unclosed visualizer adds roughly 2000 draw calls to the frame you profile
next, so the numbers you are collecting are wrong in a way that looks plausible.

**Workaround.** Close it through the Slate inspector - `Windows {action: "close", index}` -
before every subsequent measurement.

Or set `r.ProfileGPU.ShowUI 0` before profiling: the window is not created at all, and the
breakdown still goes to the Output Log.

### `stat unit` typed from the status bar does not draw over PIE

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

The command goes through, the overlay never appears, and the screenshot comes back empty.

**Workaround.** Type it into the in-game console of PIE instead (`PressKey {key: "Tilde"}`, then
`Snapshot`, then `Type` into the textbox, see the console-command note above): from there
`stat fps` and `stat unit` draw normally. To read the numbers as text, use `ProfileGPU`: it
writes a full pass breakdown into the Output Log,
which you can read with the log toolset and a pattern filter. Slower to read, but it is text,
and text is what an agent can actually use.

### After a PIE restart the Slate inspector goes blind until you `Observe` again

**Kind:** note · **Hit on:** 5.8.x (version not recorded) · **Workaround:** yes

After PIE is restarted, `Snapshot` returns five or six widgets instead of hundreds.

**Workaround.** Call `Observe` again. Observers tick about every 100 ms and are not free, so
remove them with `Unobserve` when you are done.

### `CaptureViewport` renders no particles at all

**Kind:** limitation · **Hit on:** 5.8.2 · **Workaround:** none

Neither Cascade nor Niagara shows up in a captured frame, while the same effect plays
normally in the editor viewport. Verified by dropping a known-good Niagara system at the
same spot and capturing again - still empty. Everything else in the shot renders fine,
which is what makes this expensive: the capture looks like proof that the effect is
broken, or that the asset pack does not work, and you start replacing assets.

Control Rig poses can go missing the same way (cause not established). After keys were written
to a rig, the first capture showed the pose and later ones did not, while `get_world_transform`
returned the pose and the human saw it in the editor. `refresh_sequence`, `force_evaluate` and
unlocking the camera did not help.

**No workaround.** Any visual judgement about VFX has to be made by a human looking at
the editor. Budget for that when planning an agent-driven FX pass.

A candidate, not yet tried on particles or rig poses: `EditorAppToolset.CaptureEditorImage`
captures the whole editor window, viewport included, as the human sees it (re-tested on 5.8.3).
