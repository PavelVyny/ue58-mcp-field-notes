# Unreal Engine 5.8 MCP: Field Notes

What the documentation does not tell you about driving the Unreal Editor through the native
Model Context Protocol server, and about the editor Python an agent ends up writing around it.

The official page lists five limitations, and all five are architectural: HTTP and SSE only,
loopback with no auth, Resources and Prompts not advertised, the registry adapter is
editor-only, Live Coding does not propagate new declarations. None of them is what stops you
on day two.

What stops you is a failed call that wipes the two properties it was asked to set. A create
that replaces an existing asset without asking. A script error that takes the whole editor
down, because the asset it touched happened to be open in its own window.

## The register

One file per toolset or area. Every entry: symptom in the heading, condition that triggers it,
workaround if there is one.

- [Arguments, names and paths](registry/arguments-names-and-paths.md): argument names that differ between toolsets, silent filtering in `find_*`, World Partition cells, asset paths during PIE.
- [EditorAppToolset](registry/editor-app-toolset.md): viewport capture and its camera keys, optional arguments that are required, the missing console command, GPU profiling.
- [ObjectTools](registry/object-tools.md): partial writes from a failed `set_properties`, listeners that do not rebuild, CDO delta serialization, arrays.
- [ProgrammaticToolset](registry/programmatic-toolset.md): the batching sandbox, scripts that crash the editor or roll back, `"None"` references, `_StrictDict`.
- [SequencerTools](registry/sequencer.md): cameras and cuts, section ranges and eases, play rates, Control Rig, sampling a path frame by frame, leaving a cutscene cleanly.
- [Niagara](registry/niagara.md): particle data export that delivers nothing in an editor world.
- [PCGToolset](registry/pcg.md): the toolset's real name, node data views that hang the editor, adding and updating nodes, subgraphs, pin conversions.
- [MaterialTools](registry/material-tools.md): expression inputs, graph layout, named reroutes, Material Functions that disconnect callers or crash the editor.
- [Animation and meshes](registry/animation-and-meshes.md): pointing AnimBlueprints and BlendSpaces at another skeleton, blend space axes, mesh sockets versus skeleton sockets.
- [PhysicsAssetToolset](registry/physics-asset-toolset.md): constraint reference frames, constraint motions, self-collision after `CreateFromMesh`.
- [Plugins, search and odds](registry/plugins-search-and-odds.md): plugin toggles that do not persist, semantic search, focus throttling, localised packages in the editor.
- [Blueprint graphs and editor Python](registry/blueprint-graphs-and-editor-python.md): the graph DSL, component templates, properties editor Python cannot see, FText keys, play rates in Python.

## How to read an entry

- **Kind:** `defect` (the tool does the wrong thing), `limitation` (works as designed, the
  design costs you), `note` (correct but not obvious).
- **Hit on:** the engine version where I reproduced it.
- **Workaround:** `yes`, `yes (manual)` when a human has to press something, `partial` when it
  covers only part of the problem, `n/a` for a note with nothing to work around, or `none`.

Grouped by toolset. If you know the symptom but not the toolset, search the page: the
symptom is always in the heading.

## What it covers

- The core toolsets: arguments, names and paths, `ObjectTools`, the `ProgrammaticToolset`
  batching sandbox, viewport capture.
- Sequencer work end to end: cameras and cuts, section ranges and eases, play rates, sampling
  a path frame by frame, and what decides whether the hand-off from a cutscene back to gameplay
  looks clean.
- The edges that cost the most time when you hit them: Niagara, PCG graphs, material graphs
  and functions, animation assets, physics assets.
- Editor Python where the toolsets stop: properties Python cannot see, asset creation that
  silently fails during PIE, localisation that works in a game process but never in the editor.

## Scope

Collected from July 2026 onward while building a game solo in 5.8, with an agent doing the
routine editor work. First published as one register in August, and extended since as new
entries come up in practice.

This is deliberately the toolset-specific counterpart to
[ue5-mcp](https://github.com/ibrews/ue5-mcp), a server-agnostic field manual that covers what
bites you *after* you know the tool names. This file is about the names themselves: Epic's
own toolsets, their arguments, and where they lie to you.

Reproduced in my project, on the version stated in each entry. Where I could not reproduce
something outside my own content, the entry says so. Experimental means experimental:
behaviour here can change in any release.

Not affiliated with Epic Games. One developer's log, not documentation.

## Contributing

Issues welcome: toolset, call, symptom, engine version. Corrections as much as additions. If
an entry is wrong or has gone stale, I would rather know.

## License

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
