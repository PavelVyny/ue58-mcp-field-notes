# Animation and meshes

Part of the [register](../README.md#the-register). The markers are explained in [How to read an entry](../README.md#how-to-read-an-entry).

### An AnimBlueprint or BlendSpace cannot be pointed at a different skeleton

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes (manual)

`BlendSpace.skeleton` is read-only to reflection. On an AnimBlueprint it is worse: asking for the
asset's properties gives you the CDO of the AnimInstance, so `TargetSkeleton` is not merely
unwritable, it is not visible. And `BlueprintTools.create` goes through the plain Blueprint
factory, which has no notion of a skeleton at all.

`AssetTools.duplicate` copies everything correctly, including skeleton and samples - but a
duplicate points at the same skeleton it came from, which is the one thing you were trying to
change.

**Workaround.** Create the asset by hand in the editor, on the right skeleton. That single act is
all that needs a human.

### What does work on those assets, once they exist

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** n/a

Worth stating, because the entry above reads more hopeless than it is:

- `set_parent` works on an AnimBlueprint. Reparenting a freshly created ABP onto a custom
  AnimInstance base went through normally - reparent, compile, verify with `get_parent`.
- `blendParameters` writes fine on a BlendSpace, including as a struct array, without losing
  `sampleData` or `skeleton`.

So the only things a human has to do are creating the asset on the right skeleton and authoring
the AnimGraph. Everything else can be driven.

### Renaming a blend space axis does not need a grid rebuild

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** n/a

As long as `min`, `max` and `gridNum` stay put, editing `blendParameters` is safe: the baked grid
samples store sample indices and weights, not positions. Renaming an axis or swapping which
animation a sample points at leaves the grid valid.

### `add_socket` creates a mesh socket, not a skeleton socket

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

The second argument of the underlying call is `bAddToSkeleton`, and it is false. So the
socket exists on that one mesh, and sibling meshes sharing the skeleton never see it. Verified on
disk: the socket name is present in the mesh `.uasset` and absent from its skeleton `.uasset`. Do
not check through `get_socket_names`: it returns the mesh's sockets and its skeleton's sockets in
one list.

This is convenient when you want a targeted change that does not touch a purchased pack - and
surprising if you expected sockets to live where the editor's own UI suggests they live.
