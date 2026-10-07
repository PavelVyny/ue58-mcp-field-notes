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

**→ [The register](REGISTRY.md)**

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
