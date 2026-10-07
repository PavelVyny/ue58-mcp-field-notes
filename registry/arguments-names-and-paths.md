# Arguments, names and paths

Part of the [register](../README.md#the-register). The markers are explained in [How to read an entry](../README.md#how-to-read-an-entry).

Naming across this API is not consistent, and the inconsistencies are not documented. This
section is about the tools' own surface - argument names, return shapes, silent filtering - rather than
about UE's object-path conventions in general, which
[ue5-mcp §4](https://github.com/ibrews/ue5-mcp) already covers well.

`SceneTools.trace_world` and what it returns are covered under SequencerTools, where its trap lives:
[`trace_world` returns a distance](sequencer.md#trace_world-returns-a-distance-and-a-spawnable-at-the-sampled-frame-blocks-the-ray-itself).

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
[the ProgrammaticToolset entry](programmatic-toolset.md#a-script-error-kills-the-editor-outright-when-the-edited-asset-is-open-in-its-own-window).

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
