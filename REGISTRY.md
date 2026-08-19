# The Register

Every entry: symptom in the heading, condition that triggers it, workaround if there is one.
See the [README](README.md) for what the markers mean.

## Contents

- [ProgrammaticToolset](#programmatictoolset)
- [SequencerTools](#sequencertools)
- [MaterialTools](#materialtools)
- [PhysicsAssetToolset](#physicsassettoolset)

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

---

## MaterialTools

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
