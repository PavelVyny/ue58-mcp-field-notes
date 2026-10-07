# PCGToolset

Part of the [register](../README.md#the-register). The markers are explained in [How to read an entry](../README.md#how-to-read-an-entry).

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
