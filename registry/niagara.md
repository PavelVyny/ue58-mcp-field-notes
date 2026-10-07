# Niagara

Part of the [register](../README.md#the-register). The markers are explained in [How to read an entry](../README.md#how-to-read-an-entry).

### `Export Particle Data To Blueprint` delivers nothing in an editor world

**Kind:** limitation · **Hit on:** 5.8.2 · **Workaround:** partial

In an editor world the handler is not called once: not while scrubbing a sequence that drives the
system in Desired Age mode, and not with the system looping and auto-activating in the level
viewport.

> **Correction (source-checked, 5.8.3).** An earlier version of this note said the callback queue
> (`FNiagaraWorldManager::EnqueueGlobalDeferredCallback`) is drained only on a game world tick.
> The 5.8.3 source drains the shared `GlobalDeferredCallbacks` queue in the editor too
> (`NiagaraWorldManager.cpp`). A likely cause is that `AActor::ProcessEvent` does not run
> Blueprint events on actors of an editor world unless they are `CallInEditor` (`Actor.cpp`).
> Not verified; a C++ handler may well fire in the editor.

What makes this expensive is how healthy everything looks while it happens. The stack reports zero
errors, the user parameter resolves, the handler object is bound on the component, and the log is
silent. It reads as a broken binding, and you can spend an hour re-checking the binding.

**Workaround.** Test in PIE first. The same asset works there immediately. Accept that effects
built on this interface cannot be previewed in the editor at all, and budget for tuning them in
play sessions.
