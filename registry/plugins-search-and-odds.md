# Plugins, search and odds

Part of the [register](../README.md#the-register). The markers are explained in [How to read an entry](../README.md#how-to-read-an-entry).

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

### `CompileLiveCoding` refuses instead of starting Live Coding

**Kind:** limitation · **Hit on:** 5.8.3 · **Workaround:** yes

With Live Coding disabled in Editor Preferences (the 5.8 default: `bEnabled=False` in
`BaseEditorPerProjectUserSettings.ini`), the tool returns
`Error: Live Coding is not enabled for this session` and compiles nothing. The log has no
`LogLiveCoding` lines at all.

The toolset checks `IsEnabledForSession()` before it calls `ILiveCodingModule::Compile`, although
`Compile` itself would start the session first (`EnableForSession(true)`). So an agent cannot turn
Live Coding on through this tool. By the code, `Startup = Manual` refuses the same way until Live
Coding is started by hand (not checked live).

**Workaround.** Editor Preferences → General → Live Coding → Enable Live Coding, with Startup left
on an automatic mode. After that an idle call returns `Result: NoChanges`, and an edit to a
function body in a `.cpp` returns `Result: Success` in about a second of linking. While Live
Coding is on, `Build.bat` against the running editor fails with "Unable to build while Live
Coding is active", so full builds need the editor closed.
