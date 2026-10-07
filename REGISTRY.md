# The Register

Every entry: symptom in the heading, condition that triggers it, workaround if there is one.
See the [README](README.md) for what the markers mean.

## Contents

- [Arguments, names and paths](#arguments-names-and-paths)
- [EditorAppToolset](#editorapptoolset)
- [ObjectTools](#objecttools)
- [ProgrammaticToolset](#programmatictoolset)
- [SequencerTools](#sequencertools)
- [Niagara](#niagara)
- [PCGToolset](#pcgtoolset)
- [MaterialTools](#materialtools)
- [Animation and meshes](#animation-and-meshes)
- [PhysicsAssetToolset](#physicsassettoolset)
- [Plugins, search and odds](#plugins-search-and-odds)
- [Blueprint graphs and editor Python](#blueprint-graphs-and-editor-python)

---

## Arguments, names and paths

Naming across this API is not consistent, and the inconsistencies are not documented. This
section is about the tools' own surface - argument names, return shapes, silent filtering - rather than
about UE's object-path conventions in general, which
[ue5-mcp §4](https://github.com/ibrews/ue5-mcp) already covers well.

`SceneTools.trace_world` and what it returns are covered under SequencerTools, where its trap lives:
[`trace_world` returns a distance](#trace_world-returns-a-distance-and-a-spawnable-at-the-sampled-frame-blocks-the-ray-itself).

### Argument names diverge between toolsets, and inside one toolset

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

Caught in practice, all of them by having the call fail:

- `AssetTools.delete` wants `path`. Not `asset_path`.
- `SkeletalMeshTools.set_socket_transform` wants `transform`, while `ActorTools` uses `xform`
  for the same idea.
- Inside `MaterialInstanceTools`: `list_parameters` takes `material`, and
  `get_texture_parameter` takes `instance` plus `name`. Same toolset, same kind of object, two
  different words for it.
- `AssetTools.list_folders` takes `root_path` - and its own docstring says otherwise.
- `save_assets` takes plain path strings without the `.Asset` suffix, not `{refPath}` objects,
  and there is no `assets` argument at all.

**Workaround.** Let the validation error tell you the schema - that is the intended way to
discover it. This only works when a required argument is missing: the schema is printed for a
missing required parameter. A misspelled optional argument is dropped without any error, and the
tool runs with its default.

Do the discovery outside a batch script. On 5.8.0 a wrong argument name inside a script rolled
back everything the script had already done. On 5.8.3 nothing is rolled back (re-tested live):
the calls before the failure stay applied, so rerunning the fixed script applies them twice. The
schema error itself comes back inside the script as a `RuntimeError` that `try/except` catches.

### `find_actors` does not stop at 20, but `find_assets` skips any `Actors` folder

**Kind:** defect · **Hit on:** 5.8.0, re-tested on 5.8.3 · **Workaround:** yes

> **Correction (re-tested).** An earlier version of this note said `find_actors` truncates at
> twenty results without saying so, and that `find_assets` appears to do the same. Re-tested live
> on 5.8.3: 94 actors / 26 assets returned. Neither tool has a limit parameter, and the 5.8.3
> source of both has no cap. Where the twenty came from on 5.8.0 is not established.

What `find_assets` does drop without saying so is any package whose path contains a folder named
`Actors` (or `__ExternalActors__`): a filter in the tool's own source (`asset.py`). On 5.8.3 it
returned `[]` for a plugin's `Actors` folder, while `exists` on an asset inside that folder
returned `true`.

**Workaround.** Do not read an empty `find_assets` under an `Actors` folder as absence: check the
path with `exists`.

### Actor tools only see loaded World Partition cells

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

Everything that enumerates actors sees the streamed-in part of the world only. On a partitioned
map this is a moving target - the same query gives different answers depending on where the
editor camera was.

**Workaround.** Load the region you intend to query first, and never conclude "the actor does not
exist" from an actor query alone.

### Verify an asset path before a script uses it

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

A wrong `refPath` is not the harmless "object not found" it looks like. It is the first step of
the chain that kills the editor - see
[the ProgrammaticToolset entry](#a-script-error-kills-the-editor-outright-when-the-edited-asset-is-open-in-its-own-window).

The log names it clearly once you know what to look for: `Failed to find object` → `is not valid
Object for property` → `Undo Execute tool script` → `appError`.

That chain is from 5.8.0. On 5.8.3 `execute_tool_script` runs without a transaction (re-tested
live), so the undo step that led to the crash is gone. The bad path still fails the whole script
and leaves it half-applied.

**Workaround.** `find_assets` first. Every time, including the paths you are sure about - the one
that got me was a typo in a path I had typed twenty times.

Subfolders are their own trap: plugin content does not always live where its category suggests,
and the file you want may sit one level up from the folder named after its feature.

### `exists` and `save_assets` deny that an asset exists while PIE is running

**Kind:** defect · **Hit on:** 5.8.2 · **Workaround:** yes

> **Correction (source-checked, 5.8.3).** An earlier version of this note said this starts after
> a PIE session or a C++ hot reload and lasts until a restart. The condition is narrower: it holds
> while a PIE or Simulate session is running.

`exists` answers false, and `save_assets`, `is_dirty`, `get_asset_tags` and `get_asset_class` fail
with `Asset does not exist`, for assets that plainly do. All of them go through
`EditorAssetSubsystem.DoesAssetExist`, which returns false at once while the editor is in a play
mode (`EditorScriptingHelpers.cpp`); the log has `LogUtils: Error: The Editor is currently in a
play mode` next to it. `find_assets` still returns the same path, the `.uasset` is on disk, and
`ObjectTools` reads and writes the object through its `refPath` without complaint. So edits keep
applying and only the save is lost, which is the worst shape this failure could take. Four times
in one session.

**Workaround.** `IsPIERunning`, then `StopPIE`, then save again. Treat a successful
`set_properties` made during PIE as unsaved until the save goes through.

### `save_actor` fails on a World Partition actor with "Asset does not exist"

**Kind:** defect · **Hit on:** 5.8.2 · **Workaround:** yes (manual)

`SceneTools.save_actor` answers `Asset does not exist: …/__ExternalActors__/…` for an actor
of a World Partition level. Assets save normally, the level does not. The tool routes through
`save_assets`, whose existence check turns the package path into `<package>.<package short
name>`, an object that an external-actor package does not contain. Re-tested on 5.8.3 without
PIE: `exists` on the package of an external actor whose `.uasset` is on disk returned false.

Seen on one map only. The cause is read from source; `save_actor` itself was not re-run on an
already saved actor.

**Workaround.** Save the level by hand in the editor.

---

## EditorAppToolset

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

---

## ObjectTools

### A failed `set_properties` has already written every key it could

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

> **Correction (source-checked, 5.8.3).** An earlier version of this note was titled "A failed
> `set_properties` still wipes the properties it touched".

The call is not atomic. Keys are written one by one; a rejected key (read-only, unknown, blocked)
is skipped, and the error listing the rejected keys is raised only at the end. So an error does
not mean nothing happened: every writable key in the same call is already applied, and nothing
rolls it back.

`skeleton` on an animation asset is `VisibleAnywhere` and is never written by the tool. The
`skeleton: None` / `sampleData: []` I originally saw on a BlendSpace most likely came from the
batch-script rollback of 5.8.0 (the probe ran inside a ProgrammaticToolset script), not from
`set_properties` itself. Not verified.

**Workaround.** Send one unknown property per call, and probe unknown properties on a duplicate,
never on the asset you care about.

### `set_properties` notifies the object, but not every listener rebuilds

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes (manual)

> **Correction (source-checked, 5.8.3, not re-tested live).** An earlier version of this entry
> was titled "`set_properties` never fires `PostEditChangeProperty`" and said the toolset gives
> you no way to construct the event. The 5.8.3 source contradicts it: every leaf write goes
> through `PreEditChange` and `PostEditChangeChainProperty`, always, and the base `UObject`
> forwards that to `PostEditChangeProperty` (`ToolsetLibraryImpl.cpp`, `PropertyAccessUtil.cpp`).
> Epic's own tests assert it. The 5.8.0 source was not checked.

What I saw on 5.8.0 was a system that did not rebuild after the edit. If that happens, check what
its listener filters on. Mesh Partition, for one, broadcasts `OnDefinitionModified` from
`PostEditChangeProperty`, but forces a rebuild only for a whitelist of properties
(`MeshPartitionDefinition.cpp`); for the rest you need an explicit rebuild, through the Details
panel too. Why the rebuild did not happen in my case is not explained by the source.

**Workaround.** Trigger the owning system's rebuild by hand. If a commandlet consumes the asset
afterwards, save it to disk first - a separate process does not see in-memory edits and does not
inherit console variables.

### An empty result is indistinguishable from "wrong context"

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

`[]` and `""` mean both "there is nothing here" and "the object is not loaded, or you are
asking in the wrong context". The API does not separate them, so an agent reads a clean empty
answer and concludes the collection is empty.

**Workaround.** Verify emptiness a second way before believing it. This is the single
cheapest habit on this page and the one that saves the most time.

### Delta serialization swallows a child CDO override equal to the parent value

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

Write a value onto a child Blueprint's CDO that happens to equal the parent's, and the call
reports success - but no delta is stored, because there is no difference to store. Change the
parent later, or update the plugin the parent lives in, and the child silently follows the new
parent value. The override you thought you set was never there.

The mirror image bites too: per-instance overrides on placed actors shadow CDO edits entirely.
Instances that were placed months ago keep their captured values and never see the new default.

**Workaround.** Assign the override while the parent holds a different value, and re-read after
any parent change. For stale placed actors, `reset_properties` on the single property returns
that instance to inheritance without touching the rest.

### Not every UPROPERTY is reachable through reflection

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes (manual)

`bEnableStreaming` on World Partition, for one: visible in the UI, not readable through
reflection. Absence from the reflection surface does not mean absence from the object.

**Workaround.** Change it in the editor UI and move on. Not everything is worth automating.

### `set_properties` refuses to grow an array and edit its elements in one call

**Kind:** limitation · **Hit on:** 5.8.2 · **Workaround:** yes

Send an array that is both longer than the stored one and different in its existing entries, and
the whole property is rejected: `ArrayAdd: elements changed alongside the size change; insertion
points are ambiguous`. Appending alone is fine, editing in place is fine, both at once is not.

**Workaround.** Read the current array with `get_properties`, append to what came back, and send
the old entries unchanged. Byte for byte unchanged: retyping a stored `0.78899997` as `0.789`
counts as an edit and brings the refusal back.

---

## ProgrammaticToolset

### A script error kills the editor outright when the edited asset is open in its own window

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

A script fails mid-run. The ProgrammaticToolset rolls its transaction back
(`ToolsetLibrary.undo_transaction`, a wrapper over `GEditor->UndoTransaction`). The rollback
tries to rename a preview object on top of an existing one, and the editor dies:

```
Fatal error: Renaming an object (MaterialEditorOnlyData /Engine/Transient.PreviewMaterial_88:...)
on top of an existing object (...M_<YourMaterial>EditorOnlyData) is not allowed
```

No dialog, no prompt to save. Everything unsaved is gone.

What made it fire in my case was a typo in an asset path - nothing more dramatic than that.
The chain reads clearly in the log once you know it:

```
LogUObjectGlobals: Warning: Failed to find object '...'
→ is not valid Object for property
→ LogEditorTransaction: Undo Execute tool script
→ appError
```

**Workaround.** Close the asset's own editor window before running a script that mutates it.
And resolve every asset path with `find_assets` *before* the script uses it - a bad refPath
is not a harmless "object not found", it is the entrance to this crash.

**Why it matters.** The cost of a script error is not the error. It is everything the editor
was holding.

**5.8.3 (read from source, re-tested live):** `execute_tool_script` now runs through the plain
`_ScriptRunner` (`programmatic.py:941`); the transactional runner is still in the file but
unused. There is no transaction any more: in a live re-test a script created an asset, wrote an
entry into it, then failed on a wrong argument name, and both the asset and the entry stayed. The
rollback that crashed the editor has no path to fire (not re-tested with an asset open in its own
window). Resolving paths with `find_assets` first still pays: a bad refPath fails the whole
script and leaves it half-applied.

### An error inside a script rolls back every mutation the script already made

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

The script is one transaction. A call that succeeded three steps ago is undone when a later
call fails on a wrong argument name. I watched `add_socket` complete and then physically
disappear because the next call used an argument that does not exist.

**Workaround.** Validate argument names before batching. A single wrong name costs the whole
run, not just its own step - and if the asset is open in its editor, see the entry above.

**5.8.3 (re-tested live):** nothing rolls back any more. A script that created an asset and wrote
an entry, then failed on a wrong argument name, left both in place (`execute_tool_script` runs
the plain `_ScriptRunner`, `programmatic.py:941`). A failed script is half-applied: check the
state before you rerun it, or the earlier calls run twice.

### An unset object reference comes back as the string `"None"`

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

> **Correction (re-tested).** An earlier version of this entry was titled "Subobject paths
> containing a space work in direct calls and break inside scripts". That was a wrong
> attribution: the failing call received `"None"`, not the path with the space. Re-tested on
> 5.8.3: a subobject path with a space resolved both in a direct call and inside
> `execute_tool_script`.

`get_properties` returns an empty object property as the string `"None"`, not JSON null. In a
script `d["prop"]["refPath"]` then raises `TypeError: string indices must be integers`, and
passing the value on as a refPath fails with `None is not valid value for property 'instance'`.
Subobject paths with spaces resolve the same way in direct calls and in scripts.

**Workaround.** Check `isinstance(v, dict)` before reading `refPath`.

### The dictionaries are `_StrictDict`: `.get(key, default)` raises

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

Dicts returned by `execute_tool`, and every dict nested in them, are `_StrictDict`; your own
`json.loads` gives plain dicts. `.get(key, default)` raises `TypeError: does not support a
default value` even when the key exists, and a bare `.get(key)` raises `KeyError` on a missing
key, so `.get` buys no safety. The defensive pattern everyone writes by reflex is the one that
breaks, and uncaught it aborts the whole script (on 5.8.0 that also undid everything before it;
on 5.8.3 it stays applied).

**Workaround.** `if key in d: d[key]`. Never `.get`.

### `run()` must return a dict with string keys, and the check fires after the work is done

**Kind:** limitation · **Hit on:** 5.8.2 · **Workaround:** yes

`run()` returning `{0: ..., 80: ...}` fails with `run() must return a dict[str, Any], returned
dict with non-string keys.` A list instead of a dict fails the same check. Only top-level keys
are checked: nested int keys pass, and `json.dumps` quietly turns them into strings.

The check runs after `run()` has returned, so every call the script made has already happened.
My case only read data. By the 5.8.3 source (`_ScriptRunner`, no transaction) and the live
re-test above, nothing rolls those calls back, so rerunning the whole script repeats any edits
in it.

**Workaround.** Use `str(frame)` for keys. Fix the keys and rerun only the reading part.

### `get_properties` with a property the node's class does not have kills the entire script

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

Ask for `MaterialFunction` on a node that is not a `MaterialFunctionCall` and the whole
`execute_tool_script` dies. `try/except` does not save you, not even `except BaseException`
(re-tested live on 5.8.3): the script stops at the failing call and the line after `except`
never runs. Only a schema error, such as a wrong argument name, comes back as a `RuntimeError`
that `try/except` catches.

**Workaround.** Request properties strictly by node type: `MaterialFunction` only on
`MaterialFunctionCall`, `ParameterName` only on `*Parameter` nodes, `Name` only on
`NamedRerouteDeclaration`, `R` only on `Constant`.

---

## SequencerTools

### `create_level_sequence` silently destroys an existing asset at the same path

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

Point it at a package path that already holds a Level Sequence and it does not fail, warn,
or ask. It replaces. The old sequence - tracks, keys, bindings - is gone.

Documented in the tool's own docstring in 5.8.3 (it deletes to avoid a modal overwrite dialog),
so `describe_toolset` tells you; the call itself still gives no warning.

**Workaround.** `find_assets` on the target path before every create. Treat the call as
destructive, because it is.

### A new property-track section is created with a `0..0` range, so the keys never evaluate

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

`add_section` gives you a section whose range is zero-length. Keys written into it are accepted,
`get_keys` lists them all back, and the property holds its original value on every frame except
frame 0.

The symptom is "the animation on this property does nothing", which sends you looking at the
keys - and the keys are fine.

Cause (source-checked, 5.8.3): a new section defaults to `[0, 0]` (`MovieSceneSection.cpp`). The
Sequencer editor makes new sections infinite when Infinite Key Areas is on, which it is by
default, but the scripted `add_section` does not apply that setting.

**Workaround.** `set_section_range(0, end)` immediately after every `add_section`. This is the
narrow, creation-time case of the wider rule that section ranges are independent of the
sequence playback range - see
[ue5-mcp §5.15](https://github.com/ibrews/ue5-mcp) for the general version.

The other option is `set_section_start_bounded(false)` / `set_section_end_bounded(false)`, but
`get_section_range` throws on an unbounded section (see the entry on unbounded sections below).

### A nested struct path did not animate once; cause not established

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

> **Correction (source-checked, 5.8.3).** An earlier version of this entry was titled "Property
> tracks silently ignore nested struct fields". The engine does resolve nested property paths
> (`MovieScenePropertyRegistry.cpp`), the tool's docstring suggests the dotted notation, and the
> path that worked, `FocusSettings.ManualFocusDistance`, is itself nested. The likely cause is
> the `0..0` section range from the entry above.

`set_property_name_and_path` with a path into a struct (`Filmback.SensorWidth`) created the
track, accepted the keys, and never applied the value. No error anywhere in the chain.

**Workaround.** Check the section range first. To set a whole struct once, use `set_properties`
on the spawnable instance instead of animating it.

### `create_camera` already made the property tracks, and a second one averages the values

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

`create_camera` quietly creates the standard property tracks on the child CameraComponent
binding (Current Focal Length, Manual Focus Distance, Current Aperture), keyed at the playhead
frame with the component's current values (`LevelSequenceEditorSubsystem.cpp`).
Add your own track for the same property and the engine blends two sources: the value you get
is the arithmetic mean of your keys and the original.

I asked for a focus distance of 131 cm and got 50065. That number makes no sense until you know
there are two tracks.

Because those keys sit at the playhead, a camera created with the playhead on frame 45 came with
keys on frame 45: 35 mm, f/2.8, focus 100000. That gives a second symptom: a soft image, and any
value typed into the Details panel snaps back as soon as the sequence plays, because the track
wins.

**Workaround.** `get_tracks_on_binding` plus `get_track_display_name` to find what is already
there, write into the existing track, and `remove_track` on any duplicate you created. Change the
lens by changing the key, not the component property.

### Unbounded sections are normal, and asking about their range throws

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

The transform section from `create_camera` and every spawn section are created without bounds.
That is correct - you can write keys outside a range that does not exist. But
`get_section_range` and `get_section_properties` on such a section raise "Section does not have
a start frame", and inside a batch script that single raise aborts the script (on 5.8.0 it also
rolled back everything the script had done).

**Workaround.** Check `has_section_start_frame` / `has_section_end_frame` first. `try` does
not help here - the error comes from the tool layer, not from your script.

### Changing display rate renumbers every existing key

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

Switch a sequence from 30 to 120 fps and frame 60 becomes frame 240. Absolute time is preserved,
which is the point - but every frame number you wrote down is now wrong, and code that reasons
about "the key at frame 60" quietly targets a quarter of the way through.

Related: `set_section_range` makes every section half-open, `[start, end)`
(`MovieSceneSectionExtensions.cpp`). On a Camera Cuts section that means the end has to sit at
`last_key + 1`, or the final frame is never shown through the camera.

**Workaround.** After a rate change, stop trusting your notes: read the channel with `get_keys`,
clear it, and write again from the new numbers.

### A `NaN` view range makes the Sequencer timeline disappear

**Kind:** defect · **Hit on:** 5.8.2 · **Workaround:** yes

The time ruler and every key vanish from the Sequencer panel while the tracks themselves
are intact. Track filters are empty, nothing is muted, soloed, locked or deactivated, so
the usual suspects all check out and the panel still draws nothing.

`SequencerTools.get_view_range` returns `{"start": -2.9, "end": NaN}`. The widget cannot
compute a pixel from a range whose end is not a number, so it draws none.

**Workaround.** `set_view_range` with sane seconds, then reopen the sequence - the panel
reads the range when it opens and may not pick it up live. How the value became `NaN` in
the first place was not established.

### A material parameter track cannot be built, and neither can a component binding

**Kind:** limitation · **Hit on:** 5.8.2 · **Workaround:** partial

Adding the track itself succeeds and gets you nothing: `MovieSceneComponentMaterialTrack` decides
which material slot it drives through `MaterialInfo`, a `UPROPERTY()` with no `EditAnywhere`, so
`set_properties` will not write it and the track sits there animating nothing. The section is shut
the same way - `ScalarParameterInfosAndCurves` is equally unreachable - and the function that would
create a parameter channel, `AddScalarParameterKey`, is C++ and Blueprint only, which puts it out
of reach of the sandboxed script runner as well.

Component bindings have the same shape of problem from the other side: `add_actors` takes actors,
and `rebind_component` needs a component binding to already exist, so there is no first one to
create.

**Workaround.** Both are two clicks for a person in Sequencer - add the component binding, then
pick the parameter. Everything after that is scriptable in full: once the channel exists, keys go
in through the keyframing toolset normally. Plan the flow around one short manual step rather than
trying to automate past it.

### `set_camera_cut_binding` fails on every call

**Kind:** defect · **Hit on:** 5.8.2 · **Workaround:** yes

The tool builds `unreal.Guid(camera_binding_id)` from the string you pass, and the Python `Guid`
constructor does not take a string. Every call ends in
`call() takes at most 0 arguments (1 given)`, whatever the ID.

**Workaround.** Write the property on the Camera Cut section directly:
`ObjectTools.set_properties` with
`{"CameraBindingID": {"guid": "<camera binding GUID>", "sequenceId": 0, "resolveParentIndex": 0}}`.
The GUID is the `bindingId` of the camera's binding proxy. Read it back with `get_properties` to
confirm.

### The engine's FK Control Rig cannot be added to a binding

**Kind:** limitation · **Hit on:** 5.8.2 · **Workaround:** partial

`find_or_create_track` and `bake_to_control_rig` take a path to a Control Rig asset and call
`get_control_rig_class()` on it. The built-in `FKControlRig` is a native class with no asset, so
neither tool can create it. The Mannequin rig assets are no substitute on older skeletons:
`CR_Mannequin_Body` expects the UE5 bone set (`spine_04`, `spine_05`, `neck_02`).

**Workaround.** One manual step: on the character's track, `+ Track → Control Rig → FK Control
Rig`. From then on everything is scriptable. `get_control_rigs` sees it, and the rig tools find it
by the string `"FKControlRig"`, because they match the rig by substring of its class path.

Two things save time once it is there. With no keys, the rig holds the reference pose and replaces
the animation blueprint entirely, so any bone you do not key stands in reference pose. And each
`<bone>_CONTROL` value is a delta from the reference pose in the bone's own frame:
`world = world_at_zero_delta · delta`. That model matched the editor to under 0.001 cm, and it is
enough to run your own FK and two-bone IK outside the editor and write only local values.

### `set_world_transform` on an FK control puts the bone somewhere else

**Kind:** defect · **Hit on:** 5.8.2 · **Workaround:** yes

`SequencerControlRigTools.set_world_transform` on an FK Control Rig control, asked to keep the
bone where it was and turn it to (63.4, −180, 113.4), read back as (63.4, 90, 113.4) with the bone
moved 112 cm away. The fault is in the engine, not in the tool (source-checked, 5.8.3).
`ControlRigSequencerEditorLibrary::SetControlRigWorldTransform` converts the world transform to
rig space relative to the actor's root component (the capsule), while `GetControlRigWorldTransform`
converts back through the bound skeletal mesh component. On a Character the mesh is offset from
the capsule (usually yaw −90 and Z down), so every write lands shifted by that offset: here
exactly −90 in yaw, plus the moved position. Any control is affected, not only FK, whenever the
mesh is not at the actor root. The tool's `rotation=[pitch, yaw, roll]` is fine: a list converts
to a Rotator in property order (Pitch, Yaw, Roll).

**Workaround.** Do not use it on FK controls. Compute the local delta yourself (see the entry
above) and write it with `set_euler_transform`, then check the result with `get_world_transform`,
which reads correctly.

### Reading an actor's path frame by frame: `get_actor_transform_at_frame`, not `bake_channel_keys`

**Kind:** note · **Hit on:** 5.8.3 · **Workaround:** yes

To build a camera that follows a keyframed actor you need the actor's evaluated position on every
frame, not its keys. `SequencerKeyframingTools.bake_channel_keys` looks like the tool for it and is
not: over a 346-frame range it returned a single number (it did not alter the channel's keys).
Why (source-checked, 5.8.3): it fills a `SequencerScriptingRange` with
`set_start_frame`/`set_end_frame`, and that struct's internal rate is fixed at 60000 fps, so the
frames are read as ticks (a range of a few milliseconds, less than one frame) and one value comes
back. Untested workaround: pass frames multiplied by 60000 / display rate.

**Workaround.** `SequencerControlRigTools.get_actor_transform_at_frame {sequence, actor_name, frame}`
works for any actor in the sequence, no rig required, and returns the value the sequence actually
evaluates, interpolation included. `actor_name` is matched as a substring of the label or object
name across all actors in the editor world, and the first match wins. A full name is no
guarantee: `CineCameraActor2` also matches `CineCameraActor2_5`. Take the short name of the
spawned instance from `get_bound_objects`, and check with `find_actors` that it is not part of
another actor's name or label. It returns the root component's location and rotation, no scale.
It is slow, roughly a second per call: a hundred samples plus the key writes in
one `ProgrammaticToolset` script ran past the client's 120 s and went to the background, so split
large jobs.

### `trace_world` returns a distance, and a spawnable at the sampled frame blocks the ray itself

**Kind:** limitation · **Hit on:** 5.8.3 · **Workaround:** yes

`SceneTools.trace_world {start, end}` returns only the distance to the first hit, or `null`. No hit
point, no actor. A start point inside collision returns 0. The ray uses the Visibility channel
with complex collision, and during PIE it traces the PIE world (`scene.py`). The trap: sampling a
Sequencer actor's position evaluates the sequence at that frame and leaves the world there (the
playhead itself does not move), so the spawnable now stands exactly where you are about to trace
from, and every ray hits it. A clearance check along a flight path came back as all zeros, including in
open air.

**Workaround.** Collect the positions first, then move the playhead to a frame where the spawnable
does not exist (a `false` key on its Spawn track), call `force_evaluate`, and only then trace.

### An animation section's play rate is a string inside `Params`, and setting it resizes the section

**Kind:** note · **Hit on:** 5.8.3 · **Workaround:** yes

On a skeletal animation section the rate is not a plain float:
`Params.playRate = "EMovieSceneTimeWarpType::FixedPlayRate(PlayRate=0.780000)"`, with `bReverse` in the
same struct. Read `Params` with `get_properties`, change the field, write the whole struct back with
`set_properties`. The engine then scales the section's current length by old rate / new rate,
whatever the clip length (563-660 became 563-687 at 0.78: 97 / 0.78 ≈ 124), so set the range
again afterwards. Keep the clip at least as long as the
section, or it restarts at the end and pops.

### `set_section_ease_in` / `set_section_ease_out` take ticks, not frames

**Kind:** defect · **Hit on:** 5.8.3 · **Workaround:** yes

The docstring says the duration is in frames. The tool calls
`UMovieSceneSectionEasingExtensions::SetEaseInDuration`, which writes the number straight into the
section's manual ease duration, and that field is stored in the sequence's tick resolution (24000 per
second by default). `duration: 10` is 1/40 of a frame at 60 fps, so a crossfade between two animation
sections turns into an instant cut, while every readback still reports "10". The symptom points the
wrong way: the clips look like they pop despite the ease being set.

**Workaround.** Pass `frames × tickResolution / displayRate` (400 per frame for 24000 ticks at 60 fps)
and read `Easing.manualEaseInDuration` back with `get_properties` after the write.

### A manual ease of 0 on an overlapping section pops, and script-made sections get no auto ease

**Kind:** note · **Hit on:** 5.8.3 · **Workaround:** yes

Two animation sections overlap, the incoming one eases in, and the pose still jumps on the last
frame of the outgoing one. The outgoing section had `bManualEaseOut = true` with a duration of 0:
it held full weight to its last frame and vanished, while the incoming clip was only half blended
in. The ease on the incoming section was fine; the one nobody looked at was the problem.

Clearing the manual flags in `Easing` (read the whole struct, set `bManualEaseIn` /
`bManualEaseOut` to false, write it back) switches both sections to the automatic ease, which
equals the overlap. That auto value is only computed when the sections are edited in the
Sequencer UI, though: sections created by a script with `add_section` + `set_section_range` keep
`autoEaseInDuration = 0`.

**Workaround.** Edited-by-hand sections: clear the manual flags. Script-made sections: set a
manual ease in ticks (previous note) on both sides of every overlap.

### `create_camera` makes no camera cut when the sequence already has a cut track

**Kind:** note · **Hit on:** 5.8.3 · **Workaround:** yes

The new camera gets its binding, transform and spawn tracks, and the three lens tracks on its
`CameraComponent` (see the note above), and it is placed at the viewport's pose. It gets no
section on the Camera Cuts track, so the sequence never looks through it.

**Workaround.** `add_section` on the cut track, `set_section_range`, then point the section at the
camera with `ObjectTools.set_properties` and
`{"CameraBindingID": {"guid": "<bindingId>", "sequenceId": 0, "resolveParentIndex": 0}}`
(the path from the `set_camera_cut_binding` note).

### `get_bound_objects` returns `[]` for spawned cameras that exist

**Kind:** defect · **Hit on:** 5.8.3 · **Workaround:** yes

After `set_playhead_frame`, in the same call and in the next one, `get_bound_objects` returned an
empty list for every spawnable camera, while `find_actors` showed them all in the level.
`refresh_sequence` did not help. The object names change on every respawn as well
(`CineCameraActor4_5` became `_6`), so a cached reference stops resolving.

**Workaround.** Find the spawned cameras with
`find_actors {actor_type: "/Script/CinematicCamera.CineCameraActor"}` and identify them by
`get_label`, fresh each time.

### `remove_binding` leaves an empty entry in its folder

**Kind:** defect · **Hit on:** 5.8.3 · **Workaround:** yes

A sequence folder keeps the GUID of a binding that `remove_binding` deleted, and
`get_folder_contents` lists it as a binding named `""`. There is no tool to take a child out of a
folder, and the folder's child list is not readable through `get_properties`. Child bindings
(`CameraComponent` under a camera) also stay behind unless removed first.

**Workaround.** Remove children first (`get_child_possessables`, then `remove_binding` on each).
To clean the folder, rebuild it: `add_root_folder` with the same name, `add_binding_to_folder` for
every live binding, `remove_root_folder` on the old one. Bindings are untouched by this.

### A camera cut shows the actor's label, not the binding's name

**Kind:** note · **Hit on:** 5.8.3 · **Workaround:** yes

`set_binding_name` renames the row in the Sequencer tree, but the Camera Cuts track keeps showing
`CineCameraActorN`: the cut displays the camera actor's label. `ActorTools.set_label` on the
spawned instance fixes it, and the label survives saving and reopening the sequence, so it lands
in the spawnable template.

**Workaround.** Rename both: the binding for the tree, the actor label for the cut track.

### Leaving a cutscene: sections restore by default, and `GetViewTarget` lies during the blend

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

Three things that together decide whether the hand-off from a cutscene to gameplay looks clean.

- A section whose When Finished is *Project Default* restores state: `BaseEngine.ini`,
  `[/Script/LevelSequence.LevelSequence] DefaultCompletionMode=RestoreState`. The player pawn
  driven by a transform track snaps back to where it stood before the cutscene. Set the pawn's
  transform section to Keep State.
- Blending into the gameplay camera is the ease-out of the last camera-cut section plus the
  track's Can Blend flag. The game handler then calls `SetViewTarget(pawn, blend)`.
- During that blend `APlayerCameraManager::GetViewTarget()` returns the *pending* target, the
  pawn you are blending to (`PlayerCameraManager.cpp`, "if blending to another view target,
  return this one first"). Code that wants the camera the blend is leaving must read
  `ViewTarget.Target`. We drove the control rotation from `GetViewTarget()` and the gameplay
  camera started following the pawn's yaw the moment the blend began.

**Workaround, for the hand-off to read as one shot.** The blend interpolates field of view and
position with an ease, so most of a 67° to 90° FOV change lands mid-blend and reads as a zoom, and
a static cinematic camera mixed with a gameplay camera that is already moving reads as a jerk. Make
the last cinematic camera converge on the gameplay camera itself (same FOV, same boom pose) before
the blend ends, and give the pawn the camera's yaw on the last frame, or the pawn turns towards
the control rotation when input comes back.

---

## Niagara

### `Export Particle Data To Blueprint` delivers nothing in an editor world

**Kind:** limitation · **Hit on:** 5.8.2 · **Workaround:** partial

In an editor world the handler is not called once: not while scrubbing a sequence that drives the
system in Desired Age mode, and not with the system looping and auto-activating in the level
viewport.

> **Correction (source-checked, 5.8.3).** An earlier version of this note said the callback queue
> (`FNiagaraWorldManager::EnqueueGlobalDeferredCallback`) is drained only on a game world tick.
> The 5.8.3 source drains the shared `GlobalDeferredCallbacks` queue in the editor too
> (`NiagaraWorldManager.cpp`). A likely cause is that `AActor::ProcessEvent` does not run
> Blueprint events on actors of an editor world unless they are `CallInEditor` (`Actor.cpp`).
> Not verified; a C++ handler may well fire in the editor.

What makes this expensive is how healthy everything looks while it happens. The stack reports zero
errors, the user parameter resolves, the handler object is bound on the component, and the log is
silent. It reads as a broken binding, and you can spend an hour re-checking the binding.

**Workaround.** Test in PIE first. The same asset works there immediately. Accept that effects
built on this interface cannot be previewed in the editor at all, and budget for tuning them in
play sessions.

---

## PCGToolset

### ☠️ `GetNodeDataView` hangs the editor: node count times point count decides it

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** none

Two costs. The first call turns inspection on for the component, and it never turns it off: from
then on every generation keeps the input and output data of every executed node (hundreds of
thousands of points across dozens of nodes, gigabytes of it), and the editor runs out of memory.
On top of that each call serializes the queried pin's whole output to JSON; `startIndex` /
`endIndex` only trim it after a full serialize and re-parse, and `attributeName` is the only
argument that shrinks the work. Two calls in parallel freeze it outright.

I first wrote this down as "only use it on a small test volume". That was wrong. On a graph of
about seventy nodes, a second call right after a generation froze the editor hard enough to
need a kill - **on a 40 × 40 m volume**. A small volume does not save you: the retained data is
per node.

Inspection does not turn back off. Only a restart clears it.

**Workaround.** Do not use it on a production graph at all. Debug through the PCG log with a
pattern filter, and with your eyes.

### The toolset is not called what the catalogue says

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

Its full name is `PCGToolset.PCGToolset`, and the spatial one is `PCGToolset.PCGSpatialToolset`.
Call either by the short name and you get "Toolset not found", which reads like the plugin is
missing rather than like a naming quirk. Every toolset is registered as `<Module>.<Class>`, so
this is the rule, not a PCG quirk.

### `ListNativeNodes` hides plugin nodes, `bCommonOnly: false` or not

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

The flag defaults to true, so the first listing is short. Setting it false makes the list longer -
and still without any node a plugin contributed. Those nodes exist, they are just not
discoverable through the tool that exists to discover nodes.

The cause is a filter in the tool (source-checked, 5.8.3): its node map keeps only classes from
the `/Script/PCG` package, so nodes from any other module, PCG interop modules included, are
dropped. `AddNode` looks types up in the same map, so it rejects those nodes too ("Node type …
does not exist").

**Workaround.** Find their classes by reflection (`search_subclasses` on the PCG settings base),
and build with the plugin's own primitive subgraphs through `AddSubgraphNode` rather than with
native nodes.

### Adding a native node is `AddNode`, and six of its eight arguments are required

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

There is no `AddNativeNode` - that name returns "Unknown tool". The real one is `AddNode`, and the
first six of its eight arguments are required: `graph`, `nativeNodeType`, `nodeName`,
`jsonParams`, `nodeTitle`, `nodeComment`, every one of them, every time. `xPositionIdx` and
`yPositionIdx` (default 0) are optional. The C++ declares empty-string defaults for
`jsonParams`, `nodeTitle` and `nodeComment` too, but those three are required in practice (see
"Optional arguments that are not optional"). `nativeNodeType` is the display string from `ListNativeNodes`, spaces
included: `"Spatial Noise"`, `"Density Filter"`, `"Get Spline Data"`.

### `UpdateNode` demands both `jsonParams` and `nodeTitle`, even when you change only one

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

Leave either out and the call fails; pass `""` and what is there is kept. So the argument is
required in order to be ignored.

Inside a batch script this is not a small annoyance: unless caught, the failure aborts the run
(on 5.8.0 it also rolled back every earlier mutation).

### `subGraphForNode` needs the object path with the name twice

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

`/Plugin/Primitives/Filter/Filter_Foo` is what `find_assets` gives you, and
`AddSubgraphNode` rejects it: "is not a valid object path for property 'SubGraphForNode'". It
wants `/Plugin/Primitives/Filter/Filter_Foo.Filter_Foo`.

This is the standard UE object-path convention rather than a bug, but it catches everyone,
because the tool that hands you the path hands you the form the next tool refuses.

**Workaround.** `ListAvailableSubgraphs` returns primitives already in the `/Path/X.X` form. It
only lists the folders set in PCG Toolset → Subgraph Directories (by default
`/PCGPrimitives/Primitives`), and skips paths with `Subgraphs/`, `Shared/` or `_Template_`.

### `ConnectNodePins` inserts conversion nodes, and only the return value says so

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

Connect two nodes whose data types do not line up and the tool inserts converters between them
(`FilterDataByType` and friends). Your graph now has nodes you did not add, and there is no
direct edge between the two nodes you "connected", so a later `DisconnectNodePins` fails.

The return value is the list of inserted nodes; an empty array means nothing was inserted. This
is by design: the tool's own docstring says it returns the conversion/filter nodes it added. It
is still the only notice you get.

**Workaround.** Before rewiring anything, read the actual edges from `GetGraphStructure` instead
of assuming your own connection exists.

### `paramOverrides` only holds non-default values, and that is success, not loss

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

Set a parameter to a value equal to its default and its key disappears from `paramOverrides`.
Nothing was lost - there is simply no override to record.

The trap is on the read side: a node with `mode: Perlin2D` shows no `mode` key at all, because
Perlin2D is the default. "The parameter is not set" and "the parameter is set to its default" look
identical in a dump.

**Workaround.** Read the schema with `GetNativeNodeSchema` when you need to know what a missing
key means. And guard every lookup - the dictionaries here raise on a missing key rather than
returning a default, and inside a script an uncaught raise aborts the run (on 5.8.0 it also
rolled back every earlier call).

### "Failed to call Execute" means busy, not broken

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

`ExecuteGraphInstance` refuses while the previous generation is still running, and the message
does not say so. Same message when the editor is still warming up after launch.

I originally wrote this down as an idle timeout in the client and concluded that generation had
to be triggered by hand. That was wrong too: twenty-five consecutive runs on a full-map graph
went through the tool without a single timeout. The condition is simply a pause of around fifty
seconds between runs. The fully autonomous loop - edit the graph, execute, look at the result -
does work.

The mechanism (source-checked, 5.8.3; flags read live): after a graph edit the component
regenerates by itself (`bRegenerateInEditor`, on by default), and `ExecuteGraphInstance` is
refused while that generation runs. `bGenerationInProgress` and `bGenerated` can be read with
`get_properties` on the PCG component, but the flag does not cover the gap between scheduling and
start.

**Workaround.** Poll `bGenerationInProgress` instead of waiting blind, and still retry on refusal
rather than debugging the graph.

---

## MaterialTools

### `get_expression_inputs` repeats one `output_name` when a node is fed twice by the same source

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes (manual)

When one source node is wired into several inputs of the same node, every one of those inputs
reports the output name of the first such pin. The engine function behind it,
`GetInputNodeOutputNameForMaterialExpression`, returns the output of the first input that matches
the source (`MaterialEditingLibrary.cpp`, checked on 5.8.3). It is wrong whenever those pins use
different outputs, e.g. Break -> Make MaterialAttributes: in my case all of them claimed
"Specular". A multi-output source feeding a single pin reports correctly. `input_name` is always
right.

Which means you cannot reconstruct a graph's topology from this call alone, and if you do, the
result looks coherent and is wrong.

**Workaround.** Trust `output_name` only when the source feeds one pin of that node. Otherwise
check the graph in the material editor: the toolset has no other route, because those inputs are
`UPROPERTY()` without an edit specifier and `get_properties` does not return them.

### `layout_expressions` re-lays out the entire graph, not the part you touched

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

There is no "layout selection". Call it on a graph somebody else authored and every node
moves - the grouping, the comment boxes, the deliberate spacing that made the graph readable
are all gone, and there is no undo that brings the arrangement back.

**Workaround.** Never call it on a graph you did not author. Place new nodes with explicit
`x` / `y` in `add_expression`; find the free area first by taking the maximum
`MaterialExpressionEditorY` across existing nodes.

The neighbouring `delete_unused_expressions` deserves the same caution. It removes everything
not connected to a material output - which includes the author's legacy nodes, parked
deliberately and still wanted. It reads like tidying. It is data loss.

### A named reroute cannot be traced back to its declaration

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

`NamedRerouteUsage` exposes neither `Declaration` nor `DeclarationGuid` to reflection - listing
its properties gives you editor coordinates and a description, nothing else. So when you walk
somebody else's material graph and hit a named reroute, the chain simply ends there.

**Workaround.** Infer the name from context. There is no programmatic route, so plan graph
traversal knowing it has holes.

### Rebuilding a Material Function silently disconnects every caller

**Kind:** defect · **Hit on:** 5.8.2 · **Workaround:** yes

Delete all expressions inside a Material Function and build it again - same name, same
inputs, same outputs - and every `MaterialFunctionCall` in the materials that use it
loses **all** of its input connections. The materials still compile, the log stays clean,
and nothing reports a problem.

What you see instead is a broken-looking surface. In our case an unconnected divisor
became zero, the division produced garbage on the function's gradient outputs, the
garbage fed the normal, and the material went flat and matte. The symptom points at
shading; the cause is in a different asset entirely.

**Workaround.** Edit Material Functions in place, node by node - never delete and
recreate one that has callers. After any function edit, verify with
`get_expression_inputs` on the call nodes: disconnected pins come back as `NONE`.

### The `Power` node input is called `Exp`, not `Exponent`

**Kind:** note · **Hit on:** 5.8.2 · **Workaround:** yes

`connect_expressions` fails outright when you pass `Exponent`, which is what the node
shows in the editor.

**Workaround.** Read pin names from `get_expression_input_names` rather than from the
editor label. Cheap habit, and it covers the whole node library, not just this one.

### ☠️ Adding an input to a material function with callers crashes the editor

**Kind:** defect · **Hit on:** 5.8.2 · **Workaround:** yes

`Assertion failed: MatchingInput [File: .../MaterialEditorUtilities.cpp] [Line: 635]`, and
everything unsaved goes with it. The new input changes the function signature while a
`MaterialFunctionCall` in a consuming material still carries the old pin set, and the graph rebuild
asserts on the mismatch rather than reconciling it.

This is the louder relative of the entry above about rebuilding a function: there the callers lost
their connections silently, here the process dies.

**Workaround.** Do not change the signature of a function that has consumers. If the function needs
another value, put a `CollectionParameter` node inside it and read the value from a parameter
collection - functions accept those, and the signature stays as it was.

---

## Animation and meshes

### An AnimBlueprint or BlendSpace cannot be pointed at a different skeleton

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes (manual)

`BlendSpace.skeleton` is read-only to reflection. On an AnimBlueprint it is worse: asking for the
asset's properties gives you the CDO of the AnimInstance, so `TargetSkeleton` is not merely
unwritable, it is not visible. And `BlueprintTools.create` goes through the plain Blueprint
factory, which has no notion of a skeleton at all.

`AssetTools.duplicate` copies everything correctly, including skeleton and samples - but a
duplicate points at the same skeleton it came from, which is the one thing you were trying to
change.

**Workaround.** Create the asset by hand in the editor, on the right skeleton. That single act is
all that needs a human.

### What does work on those assets, once they exist

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** n/a

Worth stating, because the entry above reads more hopeless than it is:

- `set_parent` works on an AnimBlueprint. Reparenting a freshly created ABP onto a custom
  AnimInstance base went through normally - reparent, compile, verify with `get_parent`.
- `blendParameters` writes fine on a BlendSpace, including as a struct array, without losing
  `sampleData` or `skeleton`.

So the only things a human has to do are creating the asset on the right skeleton and authoring
the AnimGraph. Everything else can be driven.

### Renaming a blend space axis does not need a grid rebuild

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** n/a

As long as `min`, `max` and `gridNum` stay put, editing `blendParameters` is safe: the baked grid
samples store sample indices and weights, not positions. Renaming an axis or swapping which
animation a sample points at leaves the grid valid.

### `add_socket` creates a mesh socket, not a skeleton socket

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

The second argument of the underlying call is `bAddToSkeleton`, and it is false. So the
socket exists on that one mesh, and sibling meshes sharing the skeleton never see it. Verified on
disk: the socket name is present in the mesh `.uasset` and absent from its skeleton `.uasset`. Do
not check through `get_socket_names`: it returns the mesh's sockets and its skeleton's sockets in
one list.

This is convenient when you want a targeted change that does not touch a purchased pack - and
surprising if you expected sockets to live where the editor's own UI suggests they live.

---

## PhysicsAssetToolset

### Constraint reference frames are not in `PhysicsAssetToolset`

**Kind:** limitation · **Hit on:** 5.8.0, re-tested on 5.8.3 · **Workaround:** yes (manual)

> **Correction (re-tested).** An earlier version of this note was titled "Constraint reference
> frames are unreachable - for reading and for writing" and said `ObjectTools` cannot walk into
> subobjects. Reading works: `get_properties` on `<PhysicsAsset>:PhysicsConstraintTemplate_0`
> with `["DefaultInstance"]` returns the whole constraint, including `constraintBone1`/`2`,
> `pos1`/`priAxis1`/`secAxis1`, `pos2`/`priAxis2`/`secAxis2`, limits and drives (5.8.3). Writing
> the frames that way was not tested.

`PhysicsAssetToolset` exposes limits (`SetConstraintLimits`), masses, shapes and modes.
It exposes nothing about Parent or Child Rotation. `get_properties(PhysicsAsset,
["ConstraintSetup"])` on the asset itself answers "could not be read".

This is worse than it sounds. Porting constraint limits from a finished character to a new
one carries **how far** a joint bends, but not **which way**. On an asset with auto-generated
frames the cone is centred on the bone, so a knee with `Swing1 Limited 65` folds forward as
happily as backward. No amount of copied numbers fixes that - the numbers are not where the
problem is.

**Workaround.** Rotate the frame by hand in the Physics Asset Editor: select the constraint →
Details → Constraint Transforms → **Parent → Rotation**, third component (Z / Yaw). Leave
Child Rotation at zero, and do not touch Parent Position - that is an offset along the bone
and it is per-skeleton.

Rule of thumb that held up across two characters: **the rotation is roughly equal to the
limit itself**. With `Swing1 Limited 65`, a yaw around 60 puts the straight leg at the edge
of the bend window, and the joint folds one way only. If you have a tuned donor asset, copy
its Rotation values directly - unlike Position, they do not depend on the mesh proportions.

### Do not compare constraint motions by their first letter

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

`Locked` and `Limited` both start with `L`. Shorten them while diffing two assets and every joint
matches, including the ones that do not. I verified a port this way once and declared it clean
when it was not.

**Workaround.** Compare full strings. It is a stupid rule and it costs nothing.

### `CreateFromMesh` builds an asset with self-collision off for every body pair

**Kind:** limitation · **Hit on:** 5.8.x (version not recorded), source-checked on 5.8.3 · **Workaround:** yes (manual)

`PhysicsAssetToolset.CreateFromMesh` uses the default `FPhysAssetCreateParams`, where
`bDisableCollisionsByDefault` is true (`PhysicsAssetUtils.h`): every new body has its collision
with every other body disabled. In Simulate the limbs pass through each other, which looks like
bad limits. The New Physics Asset dialog in the editor has the same default, but there you can
untick it; the tool has no parameter for it, and no operation on collision pairs.

**Workaround.** In the Physics Asset Editor: select all bodies → Enable Collision (all pairs),
then select all constraints → Details → Constraint Behavior → Disable Collision, which turns off
the pairs that share a joint.

---

## Plugins, search and odds

### `SetPluginEnabled` does not persist

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

It returns null, the plugin appears enabled, and after a restart it is disabled again. Nothing
was written to the `.uproject`.

The tool changes the in-memory project descriptor and only marks it dirty; it never calls
`SaveCurrentProjectToDisk`, which the Plugins browser does right after the same
`SetPluginEnabled` call (source-checked, 5.8.3).

**Workaround.** Edit the `.uproject` yourself and restart the editor.

### The semantic search toolset ships non-functional

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

`SemanticSearchToolset` is wired to OpenAI - captions and embeddings both. With no key it answers
401, and the search index on disk is empty, so the toolset that looks like the answer to "find me
the thing" is the one tool guaranteed not to work out of the box. It needs an OpenAI-compatible
endpoint. The settings say a local server (Ollama, LiteLLM) will do, and an empty key only logs a
warning, but captioning needs a vision-capable model and the `dimensions` parameter is always
sent. Untested.

**Workaround.** `find_assets`, gameplay tags, and plain text search over the project. They are
enough more often than you would expect.

### `StaticMeshTools` has no `get_mesh_info`

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

The obvious call does not exist. Closest substitutes: `get_lod_count`,
`get_triangle_count(mesh, lod)`, `get_vertex_count(mesh, lod_index)`, `get_bounds(mesh)`,
`get_lod_thresholds`, `get_material_slots`, `get_material`, `is_nanite_enabled`,
`set_nanite_enabled`. On a Nanite mesh the triangle and vertex counts come from the fallback LOD. The mesh argument is `mesh`, not `static_mesh`, and `minLOD` is not readable
through reflection at all.

Instance count on an instanced static mesh is the length of its per-instance data array - there is
no counter to ask for.

### `GetQueryDescription` reports "Empty" for editable World Conditions

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

It only describes a compiled shared definition. Point it at conditions that are still editable and
it says the query is empty - which is indistinguishable from an actually empty query.

### Client-side: agent permission classifiers block Slate clicks and PIE

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

Not an engine issue - this one is on the client. Automated approval refuses Slate clicks and
`StartPIE`, so a run that should be autonomous stops. With explicit human approval both work
exactly as documented: a click on a button reports true, and simulate mode brings a PIE world up
in about seven seconds.

**Workaround.** Ask for approval up front for the interactive parts, rather than discovering the
refusal in the middle of a sequence. And note that interactive tools take over the human's editor
while they run - always worth announcing before you do it.

### An editor without OS focus throttles to ~3 FPS, and profiles taken then are garbage

**Kind:** note · **Hit on:** 5.8.x (version not recorded) · **Workaround:** yes

With "Use Less CPU when in Background" on, an editor that is not the focused window drops to
frames of about 333 ms. A profile taken while the human is in another window (answering the
agent, for one) measures the throttle, not the game: the marker is a frame or render-thread time
of about 333 ms with an almost empty tree under it.

**Workaround.** Agree with the human to keep the editor focused for the N seconds of a
measurement.

---

### Localised packages under `/L10N/` are never used in the editor, and uncooked `-game` crashes with Mesh Partition

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** partial

Asset localisation (a copy of an asset under `Content/L10N/<culture>/` with the same relative path)
is switched off in the editor on purpose: `Linker.cpp`, "The editor must not redirect packages for
localization", guarded by `!GIsEditor`. In PIE, with any culture and any preview language, you hear
and see the source assets. It only works in a game process.

The usual way to get a game process without cooking, `UnrealEditor.exe <project> -game
-culture=xx`, dies on map load when the experimental Mesh Partition plugin is enabled:
`Assertion failed: DescriptorCache` in `MeshPartitionWorldUpdater.cpp`. The world updater asks for
an editor subsystem that does not exist without the editor.

**Workaround.** Cook. For checks in PIE, a throwaway copy of the sequence whose sections point at
the localised assets directly. Text localisation is not affected: the game-language preview works in
PIE.

## Blueprint graphs and editor Python

Everything here was hit while moving a dialogue system from a spec into a running game: adding a
component to a Blueprint from script, replacing an event graph, and importing text assets from
JSON. The tools work, but several of them fail in ways that point at the wrong cause.

### Boolean property names drop the `b` prefix in editor Python

**Kind:** note · **Hit on:** 5.8.2 · **Workaround:** yes

A `bool` UPROPERTY declared as `bIsPlayer` is reachable from Python as `is_player`, not as
`b_is_player`. The symptom misleads: the error reads `Failed to find property 'b_is_player' for
attribute 'b_is_player'`, repeating your own wrong name back at you, so it looks as if the
property is missing from the struct entirely.

**Workaround.** Strip the Hungarian `b` before converting to snake_case. Every other property
converts the way you would expect.

### `set_properties` takes `values`, `get_properties` takes `properties`

**Kind:** note · **Hit on:** 5.8.2 · **Workaround:** yes

Two neighbouring tools in the same toolset disagree about the name of the argument that carries
the property payload. The schema comes back in the error text, so a direct call self-corrects in
one round trip. Inside a batching script the failed call aborts the script unless you catch the
error; on 5.8.0 it also rolled the whole script back. On 5.8.3 nothing rolls back (re-tested
live), so the calls before it stay applied and a rerun repeats them.

**Workaround.** Validate argument names with a single direct call before putting a tool into a
batch.

### `list_properties` returns a JSON string, not a list

**Kind:** note · **Hit on:** 5.8.2 · **Workaround:** yes

The return value is a string containing a JSON object. `len()` on it gives the character count -
9325 for a widget - which reads like a plausible property count, and filtering it as if it were a
list silently finds nothing. Property names inside are camelCase, including odd ones such as
`nPC_Id` for `NPC_ID`.

**Workaround.** Parse it before using it, and print a couple of keys the first time you touch an
unfamiliar object.

### `write_graph_dsl` compiles the whole Blueprint after writing, and reports failure even though the write landed

**Kind:** limitation · **Hit on:** 5.8.2 · **Workaround:** yes

> **Correction (source-checked, 5.8.3).** An earlier version of this note said the tool compiles
> the Blueprint before it writes. The 5.8.3 source has it the other way round.

The tool writes the graph first, then compiles the whole Blueprint and raises with the errors of
every graph in it. So if another graph in the Blueprint is broken, or this graph still holds
broken events your code does not mention (the tool leaves those alone), the call fails after the
new graph is already written. It reads as "nothing changed".

**Workaround.** After an error, re-read with `read_graph_dsl` before retrying. To clear broken
nodes, `find_nodes` with an empty `title` returns every node in a graph, and `delete_node` works
without a successful compile.

### The graph DSL reader and writer are not symmetric

**Kind:** defect · **Hit on:** 5.8.2 · **Workaround:** partial

`read_graph_dsl` emits node names the writer cannot construct. Reading a getter for a public bool
member produced `|GetbIsHostile`; feeding that straight back gives `The node could not be created`.
So a graph copied out of one Blueprint is not guaranteed to write into another.

**Workaround.** Write copied graphs in pieces rather than whole, and expect to replace the
unwritable nodes with something equivalent. In our case the check belonged in C++ anyway, which is
where it ended up.

### Component template properties are lost when the Blueprint compiles

**Kind:** note · **Hit on:** 5.8.2 · **Workaround:** yes

Add a component to a Blueprint from script, set properties on its `<Name>_GEN_VARIABLE` template,
then compile, and the values are gone. No warning; the template simply looks as if nothing was
ever set on it.

**Workaround.** Compile first, set properties after. If a later step compiles again - and
`write_graph_dsl` does - set them again after that too.

### A placed instance remembers an empty override of a newly added component

**Kind:** note · **Hit on:** 5.8.2 · **Workaround:** yes

When a component is added to a Blueprint, actors already placed in a level pick it up. But an
instance that got reconstructed while the template was still empty records `None` as its own
override, and the template no longer reaches it: the component is present on the actor and its
properties read empty no matter what the Blueprint says.

**Workaround.** Restart the editor - an unsaved override dies with it and the actor takes the
template value - or set the value directly on the instance.

### The batching sandbox cannot run editor Python, and nothing else can either

**Kind:** limitation · **Hit on:** 5.8.2 · **Workaround:** yes

The batching toolset runs a sandboxed interpreter: `json`, `math`, `datetime`, `copy`, `re`,
`time`, and the tool-calling function. No `unreal`, no `os`. `open()` is swapped for a read-only
version: modes `r`/`rb`/`rt`, paths inside the project folder or its `Saved` folder only, no
writing (5.8.3, `programmatic.py`; re-tested live: a project log and the `.uproject` were read). And there is no tool
anywhere in the catalogue for running a Python file or a console command in the editor, so an
editor script cannot be launched through this API at all.

**Workaround.** Either a human runs it from the editor's Python console - `Output Log`, with the
input switched from `Cmd` to `Python` - or it runs headless as a commandlet:
`UnrealEditor-Cmd.exe <uproject> -run=pythonscript -script=<file>`.

### Localisation keys on FText from editor Python: use `unreal.NSLOCTEXT`

**Kind:** note · **Hit on:** 5.8.2 · **Workaround:** yes

> **Correction (source-checked, 5.8.3).** An earlier version of this note was titled
> "Localisation keys cannot be set on FText from editor Python" and suggested a C++ helper.

`unreal.Text.as_localizable` does not exist; `hasattr` on it returns `False`. But
`unreal.NSLOCTEXT(namespace, key, source)` does exist and calls `FText::AsLocalizable_Advanced`.
Text written without it carries no keys, and the getter returns the display string either way, so
the assets look correct. Do not check with `Text.is_culture_invariant()`: in 5.8.3 it is wired to
`IsEmptyOrWhitespace`.

**Workaround.** Build the text with `unreal.NSLOCTEXT` in the import script, and check the keys
with `KismetTextLibrary.get_text_id`.

### `UPanelSlot* Slot` shadows a member and fails the build

**Kind:** note · **Hit on:** 5.8.2 · **Workaround:** yes

Not an API problem, but it costs a build cycle: `UWidget` has a member named `Slot`, so a local
named `Slot` inside a `UUserWidget` method raises C4458, which the default project settings treat
as an error. `AddChild` returning a slot makes this a natural name to reach for.

**Workaround.** Name it anything else.

### Bare `UPROPERTY()` fields are invisible to editor Python

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

A property declared `UPROPERTY()` with no specifiers fails `get_editor_property` and
`set_editor_property` with "Failed to find property". The ones that cost us time:

- `MovieSceneCameraCutTrack.bCanBlend`: no way to set it from Python. Tick Can Blend on the track
  by hand.
- `FMovieSceneEasingSettings` fields (`ManualEaseInDuration` and the rest): read through
  `unreal.MovieSceneSectionEasingExtensions.get_ease_in_duration(section)` /
  `get_ease_out_duration`, in ticks.
- `HitResult` from `SystemLibrary.line_trace_single`: no fields at all. `hit.to_tuple()` follows
  the Break Hit Result order, `[0]` blocking hit and `[5]` impact point.
- `PlayerCameraManager` has no `get_view_target`. `PlayerController.get_view_target()` exists,
  but during a blend it returns the target being blended to.

### An animation section's play rate is a `MovieSceneTimeWarpVariant` in editor Python

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

`params.get_editor_property("play_rate")` is not a float: formatting it with `%f` raises "must be
real number, not MovieSceneTimeWarpVariant", and setting a plain number fails too. The same field is
a string through the Sequencer tools (see the SequencerTools entry above).

**Workaround.** `unreal.MovieSceneTimeWarpExtensions.conv_play_rate_to_time_warp_variant(0.7)` to
write, `to_fixed_play_rate(variant)` to read.
