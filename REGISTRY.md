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
section is about the tools' own surface - argument names, return shapes, truncation - rather than
about UE's object-path conventions in general, which
[ue5-mcp §4](https://github.com/ibrews/ue5-mcp) already covers well.

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
discover it. But do that outside a batch script: in a script a wrong argument name rolls back
everything the script has already done, so the cheap self-correction becomes an expensive one.

### `find_actors` truncates at 20 without saying so

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

Ask broadly and you get twenty results and no indication that there were more. `find_assets`
appears to do the same on wide queries.

**Workaround.** Treat exactly twenty results the way you would treat zero: as a number that means
"ask again, differently".

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

**Workaround.** `find_assets` first. Every time, including the paths you are sure about - the one
that got me was a typo in a path I had typed twenty times.

Subfolders are their own trap: plugin content does not always live where its category suggests,
and the file you want may sit one level up from the folder named after its feature.

### `exists` and `save_assets` start denying that a loaded asset exists

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** partial

After a PIE session or a C++ hot reload, `exists`, `is_dirty`, `get_asset_tags` and `save_assets`
begin answering `Asset does not exist` for assets that plainly do. `find_assets` still returns the
same path in the same call sequence, the `.uasset` is on disk, and `ObjectTools` reads and writes
the object through its `refPath` without complaint. So edits keep applying and only the save is
lost - which is the worst shape this failure could take, because nothing warns you until the next
restart throws the work away. Four times in one session.

**Workaround.** Restart the editor to clear it. Until then, ask a human to hit Save All, and treat
a successful `set_properties` as unsaved until proven otherwise.

---

## EditorAppToolset

### `CaptureViewport` ignores `captureTransform`, and the frame is not your viewport either

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** none (ask a human)

This one took three rounds to understand, so here is the whole arc.

First it looked like a stale frame: captures lagged the real viewport by several edits. The
obvious fix was to wake the viewport up - `SetCameraTransform` to the same pose, then capture.
That seemed to help, so it went in the notes as the rule.

It did not help. Next came `captureTransform`, the argument that renders from a given pose
without moving the viewport - clean, no nudging. That went in the notes too.

Then I ran three captures with three different `translation` values and got three identical
images, of a part of the level neither I nor the argument had asked for.

So: the tool returns its own fixed view. Not the active viewport, not the pose you passed.
Every explanation before this one was me fitting a story to a coincidence.

**Workaround.** None inside the toolset. For visual verification, ask the human at the keyboard
for a screenshot. `GetCameraTransform` is fine - it reports the real camera correctly, so you
can still reason about where things are. You just cannot see them.

### Optional arguments that are not optional

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

`CaptureViewport` marks `captureTransform` and `annotations` as `TOptional`. Omit either and
the call fails with `input param X needs a default value`. Both are mandatory in practice.

`StartPIE` is the same shape: it wants the full `options` block - `bSimulate`, `playMode`,
`warmupSeconds` - and `playMode` is required even when `bSimulate` already says what you mean.

Disabled annotations are not an omission, they are a block of zeros plus
`classFilter: {"refPath": ""}`.

**Workaround.** Treat `TOptional` in this API as documentation of intent, not of behaviour.
Pass everything.

### `CaptureViewport` returns ~2.8 MB and the payload lands in a file

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

The base64 image does not come back inline - it spills into a `tool-results` file, and the
client has to go read it. Worth knowing before you plan a loop around it.

The structure is `returnValue.image.data`, not `returnValue.data`. Sibling tools differ here:
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

`EditorAppToolset` can search console variables. It cannot set one, and there is no
`ExecuteConsoleCommand` anywhere in the surface. For a system built to automate the editor,
the absence is louder than most bugs on this page.

**Workaround.** Type into the editor's status-bar command box through the Slate inspector:
`Type {ref, text, submit: true}`. Find the ref with `Observe("")` then `Snapshot` on the status
bar menu - it is the textbox next to "Cmd". It works while PIE is running, too.

### Every `ProfileGPU` leaves a GPU Visualizer window open, and the next profile pays for it

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

The window is never closed. They stack up, they redraw every frame, and the editor starts
crawling - which reads as "profiling made my editor slow" rather than "I have six windows
open". Worse, an unclosed visualizer adds roughly 2000 draw calls to the frame you profile
next, so the numbers you are collecting are wrong in a way that looks plausible.

**Workaround.** Close it through the Slate inspector - `Windows {action: "close", index}` -
before every subsequent measurement.

### `stat unit` typed from the status bar does not draw over PIE

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

The command goes through, the overlay never appears, and the screenshot comes back empty.

**Workaround.** Use `ProfileGPU` instead: it writes a full pass breakdown into the Output Log,
which you can read with the log toolset and a pattern filter. Slower to read, but it is text,
and text is what an agent can actually use.

### `CaptureViewport` renders no particles at all

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** no

Neither Cascade nor Niagara shows up in a captured frame, while the same effect plays
normally in the editor viewport. Verified by dropping a known-good Niagara system at the
same spot and capturing again - still empty. Everything else in the shot renders fine,
which is what makes this expensive: the capture looks like proof that the effect is
broken, or that the asset pack does not work, and you start replacing assets.

**No workaround.** Any visual judgement about VFX has to be made by a human looking at
the editor. Budget for that when planning an agent-driven FX pass.

---

## ObjectTools

### A failed `set_properties` still wipes the properties it touched

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

I asked for `{skeleton, sampleData}` on a BlendSpace. `skeleton` is not editable, so the call
failed - and took `sampleData` with it. The asset came back with `skeleton: None` and
`sampleData: []`. Both fields I had asked about were now empty.

So the call is not atomic and it does not roll back. A rejected write is not a no-op; it is a
partial write you did not ask for.

**Workaround.** Probe unknown properties on a duplicate, never on the asset you care about.
If the duplicate comes back gutted, you have lost nothing.

### `set_properties` never fires `PostEditChangeProperty`

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes (manual)

The value changes and the asset goes dirty, so every check you can make through MCP says the
edit landed. What does not happen is the notification: systems that rebuild on
`PostEditChangeProperty` - preview meshes, generated thumbnails, anything listening on a
change delegate - never hear about it and keep serving stale state.

This is the known UE rule that a direct property write must be followed by an explicit
notify. The difference here is that you cannot do the explicit part: the toolset gives you no
way to construct the event.

**Workaround.** Touch the field once in the Details panel, or trigger the owning system's
rebuild by hand. If a commandlet consumes the asset afterwards, save it to disk first - a
separate process does not see in-memory edits and does not inherit console variables.

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

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

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

A script fails mid-run. The registry rolls the transaction back. The rollback tries to
rename a preview object on top of an existing one, and the editor dies:

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

### An error inside a script rolls back every mutation the script already made

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

The script is one transaction. A call that succeeded three steps ago is undone when a later
call fails on a wrong argument name. I watched `add_socket` complete and then physically
disappear because the next call used an argument that does not exist.

**Workaround.** Validate argument names before batching. A single wrong name costs the whole
run, not just its own step - and if the asset is open in its editor, see the entry above.

### Subobject paths containing a space work in direct calls and break inside scripts

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

A component named with a space in it - `My Component` - resolves fine through a direct
`call_tool`. Put the same path inside a batch script and `get_properties` returns `None` or
throws.

**Workaround.** Handle those objects with direct calls and keep them out of batches. Or rename
the component, if it is yours.

### The dictionaries are `_StrictDict`: `.get(key, default)` raises

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

Script-side dictionaries look like Python dicts until you call `.get()` with a fallback, which
raises `TypeError: does not support a default value`. The defensive pattern everyone writes by
reflex is the one that breaks - and it breaks the whole script, undoing everything before it.

**Workaround.** `if key in d: d[key]`. Never `.get`.

### `get_properties` with a property the node's class does not have kills the entire script

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

Ask for `MaterialFunction` on a node that is not a `MaterialFunctionCall` and the whole
`execute_tool_script` dies. `try/except` does not save you - the error comes out of
`execute_tool`, not out of Python.

**Workaround.** Request properties strictly by node type: `MaterialFunction` only on
`MaterialFunctionCall`, `ParameterName` only on `*Parameter` nodes, `Name` only on
`NamedRerouteDeclaration`, `R` only on `Constant`.

---

## SequencerTools

### `create_level_sequence` silently destroys an existing asset at the same path

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

Point it at a package path that already holds a Level Sequence and it does not fail, warn,
or ask. It replaces. The old sequence - tracks, keys, bindings - is gone.

**Workaround.** `find_assets` on the target path before every create. Treat the call as
destructive, because it is.

### A new property-track section is created with a `0..0` range, so the keys never evaluate

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

`add_section` gives you a section whose range is zero-length. Keys written into it are accepted,
`get_keys` lists them all back, and the property holds its original value on every frame except
frame 0.

The symptom is "the animation on this property does nothing", which sends you looking at the
keys - and the keys are fine.

**Workaround.** `set_section_range(0, end)` immediately after every `add_section`. This is the
narrow, creation-time case of the wider rule that section ranges are independent of the
sequence playback range - see
[ue5-mcp §5.15](https://github.com/ibrews/ue5-mcp) for the general version.

### Property tracks silently ignore nested struct fields

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

`set_property_name_and_path` with a path into a struct - `Filmback.SensorWidth` - creates the
track, accepts the keys, and never applies the value. No error anywhere in the chain.

Flat properties work (`CurrentFocalLength`, `bConstrainAspectRatio`), and so do paths the engine
itself registers (`FocusSettings.ManualFocusDistance`). Arbitrary struct paths do not.

**Workaround.** Set whole structs through `set_properties` on the spawnable instance instead of
animating them. If you need the struct field to change over time, find the engine-registered
path for it or animate something else.

### `create_camera` already made the property tracks, and a second one averages the values

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

`create_camera` quietly creates the standard property tracks on the child CameraComponent
binding - Current Focal Length, Manual Focus Distance, Current Aperture - with empty sections.
Add your own track for the same property and the engine blends two sources: the value you get
is the arithmetic mean of your keys and the original.

I asked for a focus distance of 131 cm and got 50065. That number makes no sense until you know
there are two tracks.

The sections are not always empty. A camera created with the playhead on frame 45 came with keys
on frame 45: 35 mm, f/2.8, focus 100000. The symptom then is different - a soft image, and any
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
a start frame", and inside a batch script that single raise rolls back everything the script has
done.

**Workaround.** Do not call range queries on sections you did not explicitly bound. `try` does
not help here - the error comes from the tool layer, not from your script.

### Changing display rate renumbers every existing key

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

Switch a sequence from 30 to 120 fps and frame 60 becomes frame 240. Absolute time is preserved,
which is the point - but every frame number you wrote down is now wrong, and code that reasons
about "the key at frame 60" quietly targets a quarter of the way through.

Related: the Camera Cuts section is half-open. Its end has to sit at `last_key + 1`, or the final
frame is never shown through the camera.

**Workaround.** After a rate change, stop trusting your notes: read the channel with `get_keys`,
clear it, and write again from the new numbers.

### A `NaN` view range makes the Sequencer timeline disappear

**Kind:** bug · **Hit on:** 5.8.0 · **Workaround:** yes

The time ruler and every key vanish from the Sequencer panel while the tracks themselves
are intact. Track filters are empty, nothing is muted, soloed, locked or deactivated, so
the usual suspects all check out and the panel still draws nothing.

`SequencerTools.get_view_range` returns `{"start": -2.9, "end": NaN}`. The widget cannot
compute a pixel from a range whose end is not a number, so it draws none.

**Workaround.** `set_view_range` with sane seconds, then reopen the sequence - the panel
reads the range when it opens and may not pick it up live. How the value became `NaN` in
the first place was not established.

### A material parameter track cannot be built, and neither can a component binding

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** partial

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

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

The tool builds `unreal.Guid(camera_binding_id)` from the string you pass, and the Python `Guid`
constructor does not take a string. Every call ends in
`call() takes at most 0 arguments (1 given)`, whatever the ID.

**Workaround.** Write the property on the Camera Cut section directly:
`ObjectTools.set_properties` with
`{"CameraBindingID": {"guid": "<camera binding GUID>", "sequenceId": 0, "resolveParentIndex": 0}}`.
The GUID is the `bindingId` of the camera's binding proxy. Read it back with `get_properties` to
confirm.

### The engine's FK Control Rig cannot be added to a binding

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** partial

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

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

`SequencerControlRigTools.set_world_transform` on an FK Control Rig control, asked to keep the
bone where it was and turn it to (63.4, −180, 113.4), read back as (63.4, 90, 113.4) with the bone
moved 112 cm away. The cause is not established. The tool builds `unreal.Transform(rotation=[pitch,
yaw, roll])`, so component order is the first suspect, but order alone does not explain the moved
position.

**Workaround.** Do not use it on FK controls. Compute the local delta yourself (see the entry
above) and write it with `set_euler_transform`, then check the result with `get_world_transform`,
which reads correctly.

---

## Niagara

### `Export Particle Data To Blueprint` delivers nothing in an editor world

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** partial

The data interface does not call your handler directly. Particles go into a queue, and
`PerInstanceTickPostSimulate` hands that queue to `FNiagaraWorldManager::EnqueueGlobalDeferredCallback`,
which is drained on a game world tick. In the editor it is never drained, so the handler is not
called once - not while scrubbing a sequence that drives the system in Desired Age mode, and not
with the system looping and auto-activating in the level viewport.

What makes this expensive is how healthy everything looks while it happens. The stack reports zero
errors, the user parameter resolves, the handler object is bound on the component, and the log is
silent. It reads as a broken binding, and you can spend an hour re-checking the binding.

**Workaround.** Test in PIE first. The same asset works there immediately. Accept that effects
built on this interface cannot be previewed in the editor at all, and budget for tuning them in
play sessions.

---

## PCGToolset

### ☠️ `GetNodeDataView` hangs the editor, and graph size is what decides it

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** none

The tool turns on graph-level data inspection. From then on every generation retains the data
of every node - hundreds of thousands of points across dozens of nodes, gigabytes of it - and
the editor runs out of memory. Two calls in parallel freeze it outright.

I first wrote this down as "only use it on a small test volume". That was wrong. On a graph of
about seventy nodes, a second call right after a generation froze the editor hard enough to
need a kill - **on a 40 × 40 m volume**. The volume is not what costs you. The graph is.

Inspection does not turn back off. Only a restart clears it.

**Workaround.** Do not use it on a production graph at all. Debug through the PCG log with a
pattern filter, and with your eyes.

### The toolset is not called what the catalogue says

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

Its full name is `PCGToolset.PCGToolset`, and the spatial one is `PCGToolset.PCGSpatialToolset`.
Call either by the short name and you get "Toolset not found", which reads like the plugin is
missing rather than like a naming quirk.

### `ListNativeNodes` hides plugin nodes, `bCommonOnly: false` or not

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

The flag defaults to true, so the first listing is short. Setting it false makes the list longer -
and still without any node a plugin contributed. Those nodes exist, they are just not
discoverable through the tool that exists to discover nodes.

**Workaround.** Find their classes by reflection (`search_subclasses` on the PCG settings base),
and build with the plugin's own primitive subgraphs through `AddSubgraphNode` rather than with
native nodes.

### Adding a native node is `AddNode`, and all six arguments are required

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

There is no `AddNativeNode` - that name returns "Unknown tool". The real one is `AddNode`, and it
wants `graph`, `nativeNodeType`, `nodeName`, `jsonParams`, `nodeTitle`, `nodeComment`, every one
of them, every time. `nativeNodeType` is the display string from `ListNativeNodes`, spaces
included: `"Spatial Noise"`, `"Density Filter"`, `"Get Spline Data"`.

### `UpdateNode` demands `nodeTitle` even when you are only changing parameters

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

Leave it out and the call fails; pass `""` and the existing title is kept. So the argument is
required in order to be ignored.

Inside a batch script this is not a small annoyance - the failure rolls back every mutation the
script has already made.

### `subGraphForNode` needs the object path with the name twice

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

`/Plugin/Primitives/Filter/Filter_Foo` is what `find_assets` gives you, and
`AddSubgraphNode` rejects it: "is not a valid object path for property 'SubGraphForNode'". It
wants `/Plugin/Primitives/Filter/Filter_Foo.Filter_Foo`.

This is the standard UE object-path convention rather than a bug, but it catches everyone,
because the tool that hands you the path hands you the form the next tool refuses.

### `ConnectNodePins` silently inserts conversion nodes

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

Connect two nodes whose data types do not line up and the tool inserts converters between them -
`FilterDataByType` and friends - without telling you. Your graph now has nodes you did not add,
and there is no direct edge between the two nodes you "connected", so a later
`DisconnectNodePins` fails.

The return value is the list of inserted nodes; an empty array means nothing was inserted. That
is the only notice you get.

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
returning a default, and inside a script that raise costs you the whole run.

### "Failed to call Execute" means busy, not broken

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

`ExecuteGraphInstance` refuses while the previous generation is still running, and the message
does not say so. Same message when the editor is still warming up after launch.

I originally wrote this down as an idle timeout in the client and concluded that generation had
to be triggered by hand. That was wrong too: twenty-five consecutive runs on a full-map graph
went through the tool without a single timeout. The condition is simply a pause of around fifty
seconds between runs. The fully autonomous loop - edit the graph, execute, look at the result -
does work.

**Workaround.** Wait and retry rather than debugging the graph.

---

## MaterialTools

### `get_expression_inputs` reports the wrong `output_name` on multi-output nodes

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

When the source node has several outputs - a break-attributes node, for instance - every
connection comes back naming the same output. In my case all of them claimed to come from
"Specular". The `input_name` side is correct; it is only the output that lies.

Which means you cannot reconstruct a graph's topology from this call alone, and if you do, the
result looks coherent and is wrong.

**Workaround.** Cross-check with the node's real output names before believing any edge that
starts at a multi-output node.

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

**Kind:** bug · **Hit on:** 5.8.0 · **Workaround:** yes

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

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

`connect_expressions` fails outright when you pass `Exponent`, which is what the node
shows in the editor.

**Workaround.** Read pin names from `get_expression_input_names` rather than from the
editor label. Cheap habit, and it covers the whole node library, not just this one.

### ☠️ Adding an input to a material function with callers crashes the editor

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

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

The second argument of the underlying call is `bAddSocketToSkeleton`, and it is false. So the
socket exists on that one mesh, and sibling meshes sharing the skeleton never see it. Verified
the unhappy way: a socket added to one variant of a character was simply absent on the other.

This is convenient when you want a targeted change that does not touch a purchased pack - and
surprising if you expected sockets to live where the editor's own UI suggests they live.

---

## PhysicsAssetToolset

### Constraint reference frames are unreachable - for reading and for writing

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes (manual)

`PhysicsAssetToolset` exposes limits (`SetConstraintLimits`), masses, shapes and modes.
It exposes nothing about Parent or Child Rotation. Reflection is closed too:
`get_properties(PhysicsAsset, ["ConstraintSetup"])` answers "could not be read", and
`ObjectTools` has no way to walk into subobjects.

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

---

## Plugins, search and odds

### `SetPluginEnabled` does not persist

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

It returns null, the plugin appears enabled, and after a restart it is disabled again. Nothing
was written to the `.uproject`.

**Workaround.** Edit the `.uproject` yourself and restart the editor.

### The semantic search toolset ships non-functional

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

`SemanticSearchToolset` is wired to OpenAI - captions and embeddings both. With no key it answers
401, and the search index on disk is empty, so the toolset that looks like the answer to "find me
the thing" is the one tool guaranteed not to work out of the box. Making it work costs money at a
third party.

**Workaround.** `find_assets`, gameplay tags, and plain text search over the project. They are
enough more often than you would expect.

### `StaticMeshTools` has no `get_mesh_info`

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

The obvious call does not exist. What exists: `get_lod_count`, `get_triangle_count(mesh, lod)`,
`get_lod_thresholds`, `get_material_slots`, `get_material`, `is_nanite_enabled`,
`set_nanite_enabled`. The mesh argument is `mesh`, not `static_mesh`, and `minLOD` is not readable
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

---

## Blueprint graphs and editor Python

Everything here was hit while moving a dialogue system from a spec into a running game: adding a
component to a Blueprint from script, replacing an event graph, and importing text assets from
JSON. The tools work, but several of them fail in ways that point at the wrong cause.

### Boolean property names drop the `b` prefix in editor Python

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

A `bool` UPROPERTY declared as `bIsPlayer` is reachable from Python as `is_player`, not as
`b_is_player`. The symptom misleads: the error reads `Failed to find property 'b_is_player' for
attribute 'b_is_player'`, repeating your own wrong name back at you, so it looks as if the
property is missing from the struct entirely.

**Workaround.** Strip the Hungarian `b` before converting to snake_case. Every other property
converts the way you would expect.

### `set_properties` takes `values`, `get_properties` takes `properties`

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

Two neighbouring tools in the same toolset disagree about the name of the argument that carries
the property payload. The schema comes back in the error text, so a direct call self-corrects in
one round trip - but inside a batching script the failed call rolls the whole script back, and
everything it had already done is undone with it.

**Workaround.** Validate argument names with a single direct call before putting a tool into a
batch.

### `list_properties` returns a JSON string, not a list

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

The return value is a string containing a JSON object. `len()` on it gives the character count -
9325 for a widget - which reads like a plausible property count, and filtering it as if it were a
list silently finds nothing. Property names inside are camelCase, including odd ones such as
`nPC_Id` for `NPC_ID`.

**Workaround.** Parse it before using it, and print a couple of keys the first time you touch an
unfamiliar object.

### `write_graph_dsl` compiles the Blueprint before it writes

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

Which means you cannot use it to repair a graph that does not currently compile: to replace the
broken graph you must first make it compile. The failure surfaces as a compile error listing
problems in the graph you were about to delete.

**Workaround.** `find_nodes` with an empty `title` returns every node in a graph, and
`delete_node` works without a successful compile. Delete first, then write.

### The graph DSL reader and writer are not symmetric

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** partial

`read_graph_dsl` emits node names the writer cannot construct. Reading a getter for a public bool
member produced `|GetbIsHostile`; feeding that straight back gives `The node could not be created`.
So a graph copied out of one Blueprint is not guaranteed to write into another.

**Workaround.** Write copied graphs in pieces rather than whole, and expect to replace the
unwritable nodes with something equivalent. In our case the check belonged in C++ anyway, which is
where it ended up.

### Component template properties are lost when the Blueprint compiles

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

Add a component to a Blueprint from script, set properties on its `<Name>_GEN_VARIABLE` template,
then compile, and the values are gone. No warning; the template simply looks as if nothing was
ever set on it.

**Workaround.** Compile first, set properties after. If a later step compiles again - and
`write_graph_dsl` does - set them again after that too.

### A placed instance remembers an empty override of a newly added component

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

When a component is added to a Blueprint, actors already placed in a level pick it up. But an
instance that got reconstructed while the template was still empty records `None` as its own
override, and the template no longer reaches it: the component is present on the actor and its
properties read empty no matter what the Blueprint says.

**Workaround.** Restart the editor - an unsaved override dies with it and the actor takes the
template value - or set the value directly on the instance.

### The batching sandbox cannot run editor Python, and nothing else can either

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

The batching toolset runs a sandboxed interpreter: `json`, `math`, `datetime`, `copy`, `re`,
`time`, and the tool-calling function. No `unreal`, no `os`, no file access. And there is no tool
anywhere in the catalogue for running a Python file or a console command in the editor, so an
editor script cannot be launched through this API at all.

**Workaround.** Either a human runs it from the editor's Python console - `Output Log`, with the
input switched from `Cmd` to `Python` - or it runs headless as a commandlet:
`UnrealEditor-Cmd.exe <uproject> -run=pythonscript -script=<file>`.

### Localisation keys cannot be set on FText from editor Python

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

`unreal.Text.as_localizable` does not exist; `hasattr` on it returns `False`. Any text an import
script writes into an asset is culture-invariant, and nothing shows it: the getter returns the
display string either way, so the assets look correct while carrying no keys at all.

**Workaround.** A one-function `UBlueprintFunctionLibrary` over `FText::AsLocalizable_Advanced`,
called from the import script. Worth checking with `hasattr` before assuming any Python-side text
helper exists - several documented ones do not.

### `UPanelSlot* Slot` shadows a member and fails the build

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

Not an API problem, but it costs a build cycle: `UWidget` has a member named `Slot`, so a local
named `Slot` inside a `UUserWidget` method raises C4458, which the default project settings treat
as an error. `AddChild` returning a slot makes this a natural name to reach for.

**Workaround.** Name it anything else.
