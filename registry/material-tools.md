# MaterialTools

Part of the [register](../README.md#the-register). The markers are explained in [How to read an entry](../README.md#how-to-read-an-entry).

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
