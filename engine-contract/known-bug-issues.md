# Tracking issues for `known-bug` differences

`quickjs-difference-allowlist.toml` allows a `known-bug` entry to keep the gate green, but only when
the entry names the issue that tracks the defect. A single issue cannot honestly track unrelated
shim defects, so each entry names its own.

The entries below are real shim defects rather than capability differences: the entry point exists
on both engines, and QuickJS returns a different answer instead of refusing the feature. The text is
ready to file; replace the `issue` field of the matching allowlist entry with the new URL.

File them in <https://github.com/openharmony-rs/servo-quickjs/issues>.

---

## 1. `CustomAutoRooter` trace callbacks never run during an explicit collection

- Case: `rooter.trace_on_gc`
- Reproduce: `run.sh run-spidermonkey-contract --case rooter.trace_on_gc` passes; the QuickJS run
  records `the custom rooter trace callback did not run during an explicit collection`.
- Cause: `JS_GC` in `mozjs/src/engine/gc.rs` does not walk `CustomAutoRooter` instances, so a
  rooter's `trace` hook is never invoked.

## 2. `JS_SetGCCallback`, `SetGCSliceCallback`, `JS_AddExtraGCRootsTracer` and `JS_Get/SetGCParameter` are silent no-ops

- Case: `misc.gc_callbacks`
- Cause: the shim implements those entry points as empty bodies. Registered callbacks and extra-root
  tracers never run during `JS_GC`, `SetGCSliceCallback` reports itself as already installed, and GC
  parameters read back as `0`.

## 3. `JS_RemoveRawValueRoot` leaves an extra reference, so the object survives the next `JS_GC`

- Case: `misc.roots_options_memory` (step `raw_root`)
- Cause: the raw-value root's release path duplicates the value and never drops the original
  reference, so the referent is not finalized.

## 4. The global trace hook runs twice for one explicit collection

- Case: `realm.enter_exit_and_private`
- Expected: `global_trace_hook_calls = 1` (SpiderMonkey); QuickJS records `2`. The trace hook is
  registered for both the realm and its global.

## 5. `JS_FireOnNewGlobalObject` never calls the debugger hook

- Case: `debugger.new_global_object_hook`
- Cause: the debugger emulation declares `Debugger.onNewGlobalObject` but `JS_FireOnNewGlobalObject`
  does not dispatch it, so the hook fires 0 times instead of 1.

## 6. `JS_NewEnumerateStandardClasses`, `JS_ResolveStandardClass` and `JS_MayResolveStandardClass` are no-ops

- Case: `misc.standard_classes`
- Cause: the standard-class hooks are empty, so lazily resolved class names are never enumerated and
  a name is never resolved through the global's class ops.

## 7. `HasAnyMicroTasks` does not see promise reactions queued by the engine

- Case: `promise.internal_job_queue`
- Cause: `AddPromiseReactions` enqueues into QuickJS's own job list, while `HasAnyMicroTasks` only
  reports the shim's internal queue. `RunJobs` still drains everything, so the queue is not lost,
  only invisible.

## 8. `CallObjectRootTracer`, `CallValueRootTracer`, `CallPropertyDescriptorTracer` and `TraceValueArray` do nothing

- Case: `class.edge_tracers`
- Cause: those glue entry points are bound to no-op bodies. SpiderMonkey keeps the four raw-slot
  referents alive while tracing and collects them when tracing is disabled; QuickJS finalizes none
  of them in either pass because the autorelease pool holds the children for the whole task.

## 9. `JS_GetGCParameter`/`JS_SetGCParameter` do not persist the parameter values

- Case: `runtime.context_lifecycle`
- Cause: the setters store nothing and the getters return `0`, so a read-back after a write reports
  `0` instead of the value that was set.

---

## Entries that keep issue 57

`eval.environment_chain`, `edge.eval.compile_and_environment`, `edge.weakmap`, `edge.weakmap_overwrite`,
`objects.weakmap_wpt_get_entry` and `objects.weakmap_wpt_set_entry` stay on
<https://github.com/openharmony-rs/servo-ohos/issues/57>, which covers exactly those two defects: the
environment-chain append order and the holder-map key type.

`streaming.consume_callback` is not a defect at all. QuickJS has no WebAssembly, so it has no stream
consumer; the entry is now `accepted` instead of `known-bug`.

`errors.core` is also `accepted`, but one of its steps shows a separate shim defect: QuickJS records
`JS_GetScriptedCallerPrivate` as `None` for a frame that has a private value, where SpiderMonkey
records `Some(77)`. That deserves its own `known-bug` entry (and issue) once the step is split out of
the compound case.
