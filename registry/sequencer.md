# SequencerTools

Part of the [register](../README.md#the-register). The markers are explained in [How to read an entry](../README.md#how-to-read-an-entry).

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
