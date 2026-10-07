# Blueprint graphs and editor Python

Part of the [register](../README.md#the-register). The markers are explained in [How to read an entry](../README.md#how-to-read-an-entry).

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
a string through the Sequencer tools (see [the play-rate entry](sequencer.md#an-animation-sections-play-rate-is-a-string-inside-params-and-setting-it-resizes-the-section)).

**Workaround.** `unreal.MovieSceneTimeWarpExtensions.conv_play_rate_to_time_warp_variant(0.7)` to
write, `to_fixed_play_rate(variant)` to read.
