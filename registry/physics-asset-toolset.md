# PhysicsAssetToolset

Part of the [register](../README.md#the-register). The markers are explained in [How to read an entry](../README.md#how-to-read-an-entry).

### Constraint reference frames are not in `PhysicsAssetToolset`

**Kind:** limitation · **Hit on:** 5.8.0, re-tested on 5.8.3 · **Workaround:** yes (manual)

> **Correction (re-tested).** An earlier version of this note was titled "Constraint reference
> frames are unreachable - for reading and for writing" and said `ObjectTools` cannot walk into
> subobjects. Reading works: `get_properties` on `<PhysicsAsset>:PhysicsConstraintTemplate_0`
> with `["DefaultInstance"]` returns the whole constraint, including `constraintBone1`/`2`,
> `pos1`/`priAxis1`/`secAxis1`, `pos2`/`priAxis2`/`secAxis2`, limits and drives (5.8.3). Writing
> the frames that way was not tested.

`PhysicsAssetToolset` exposes limits (`SetConstraintLimits`), masses, shapes and modes.
It exposes nothing about Parent or Child Rotation. `get_properties(PhysicsAsset,
["ConstraintSetup"])` on the asset itself answers "could not be read".

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

### Do not compare constraint motions by their first letter

**Kind:** note · **Hit on:** 5.8.0 · **Workaround:** yes

`Locked` and `Limited` both start with `L`. Shorten them while diffing two assets and every joint
matches, including the ones that do not. I verified a port this way once and declared it clean
when it was not.

**Workaround.** Compare full strings. It is a stupid rule and it costs nothing.

### `CreateFromMesh` builds an asset with self-collision off for every body pair

**Kind:** limitation · **Hit on:** 5.8.x (version not recorded), source-checked on 5.8.3 · **Workaround:** yes (manual)

`PhysicsAssetToolset.CreateFromMesh` uses the default `FPhysAssetCreateParams`, where
`bDisableCollisionsByDefault` is true (`PhysicsAssetUtils.h`): every new body has its collision
with every other body disabled. In Simulate the limbs pass through each other, which looks like
bad limits. The New Physics Asset dialog in the editor has the same default, but there you can
untick it; the tool has no parameter for it, and no operation on collision pairs.

**Workaround.** In the Physics Asset Editor: select all bodies → Enable Collision (all pairs),
then select all constraints → Details → Constraint Behavior → Disable Collision, which turns off
the pairs that share a joint.
