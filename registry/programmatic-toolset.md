# ProgrammaticToolset

Part of the [register](../README.md#the-register). The markers are explained in [How to read an entry](../README.md#how-to-read-an-entry).

### A script error kills the editor outright when the edited asset is open in its own window

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

A script fails mid-run. The ProgrammaticToolset rolls its transaction back
(`ToolsetLibrary.undo_transaction`, a wrapper over `GEditor->UndoTransaction`). The rollback
tries to rename a preview object on top of an existing one, and the editor dies:

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

**5.8.3 (read from source, re-tested live):** `execute_tool_script` now runs through the plain
`_ScriptRunner` (`programmatic.py:941`); the transactional runner is still in the file but
unused. There is no transaction any more: in a live re-test a script created an asset, wrote an
entry into it, then failed on a wrong argument name, and both the asset and the entry stayed. The
rollback that crashed the editor has no path to fire (not re-tested with an asset open in its own
window). Resolving paths with `find_assets` first still pays: a bad refPath fails the whole
script and leaves it half-applied.

### An error inside a script rolls back every mutation the script already made

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

The script is one transaction. A call that succeeded three steps ago is undone when a later
call fails on a wrong argument name. I watched `add_socket` complete and then physically
disappear because the next call used an argument that does not exist.

**Workaround.** Validate argument names before batching. A single wrong name costs the whole
run, not just its own step - and if the asset is open in its editor, see the entry above.

**5.8.3 (re-tested live):** nothing rolls back any more. A script that created an asset and wrote
an entry, then failed on a wrong argument name, left both in place (`execute_tool_script` runs
the plain `_ScriptRunner`, `programmatic.py:941`). A failed script is half-applied: check the
state before you rerun it, or the earlier calls run twice.

### An unset object reference comes back as the string `"None"`

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

> **Correction (re-tested).** An earlier version of this entry was titled "Subobject paths
> containing a space work in direct calls and break inside scripts". That was a wrong
> attribution: the failing call received `"None"`, not the path with the space. Re-tested on
> 5.8.3: a subobject path with a space resolved both in a direct call and inside
> `execute_tool_script`.

`get_properties` returns an empty object property as the string `"None"`, not JSON null. In a
script `d["prop"]["refPath"]` then raises `TypeError: string indices must be integers`, and
passing the value on as a refPath fails with `None is not valid value for property 'instance'`.
Subobject paths with spaces resolve the same way in direct calls and in scripts.

**Workaround.** Check `isinstance(v, dict)` before reading `refPath`.

### The dictionaries are `_StrictDict`: `.get(key, default)` raises

**Kind:** limitation · **Hit on:** 5.8.0 · **Workaround:** yes

Dicts returned by `execute_tool`, and every dict nested in them, are `_StrictDict`; your own
`json.loads` gives plain dicts. `.get(key, default)` raises `TypeError: does not support a
default value` even when the key exists, and a bare `.get(key)` raises `KeyError` on a missing
key, so `.get` buys no safety. The defensive pattern everyone writes by reflex is the one that
breaks, and uncaught it aborts the whole script (on 5.8.0 that also undid everything before it;
on 5.8.3 it stays applied).

**Workaround.** `if key in d: d[key]`. Never `.get`.

### `run()` must return a dict with string keys, and the check fires after the work is done

**Kind:** limitation · **Hit on:** 5.8.2 · **Workaround:** yes

`run()` returning `{0: ..., 80: ...}` fails with `run() must return a dict[str, Any], returned
dict with non-string keys.` A list instead of a dict fails the same check. Only top-level keys
are checked: nested int keys pass, and `json.dumps` quietly turns them into strings.

The check runs after `run()` has returned, so every call the script made has already happened.
My case only read data. By the 5.8.3 source (`_ScriptRunner`, no transaction) and the live
re-test above, nothing rolls those calls back, so rerunning the whole script repeats any edits
in it.

**Workaround.** Use `str(frame)` for keys. Fix the keys and rerun only the reading part.

### `get_properties` with a property the node's class does not have kills the entire script

**Kind:** defect · **Hit on:** 5.8.0 · **Workaround:** yes

Ask for `MaterialFunction` on a node that is not a `MaterialFunctionCall` and the whole
`execute_tool_script` dies. `try/except` does not save you, not even `except BaseException`
(re-tested live on 5.8.3): the script stops at the failing call and the line after `except`
never runs. Only a schema error, such as a wrong argument name, comes back as a `RuntimeError`
that `try/except` catches.

**Workaround.** Request properties strictly by node type: `MaterialFunction` only on
`MaterialFunctionCall`, `ParameterName` only on `*Parameter` nodes, `Name` only on
`NamedRerouteDeclaration`, `R` only on `Constant`.
