# ObjectTools

Part of the [register](../README.md#the-register). The markers are explained in [How to read an entry](../README.md#how-to-read-an-entry).

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
