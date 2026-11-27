# Tracking issues for `known-bug` differences

`quickjs-difference-allowlist.toml` allows a `known-bug` entry to keep the gate green, but only when
the entry names the issue that tracks the defect. A single issue cannot honestly track unrelated
shim defects, so each entry names the issue whose scope covers it.

File defects in <https://github.com/openharmony-rs/servo-quickjs/issues>.

| Issue | Scope | Allowlist entries |
| --- | --- | --- |
| [#24](https://github.com/openharmony-rs/servo-quickjs/issues/24) | `JS_GC` does not walk `CustomAutoRooter`; the GC hook and tracer entry points are not wired | `rooter.trace_on_gc`, `misc.gc_callbacks`, `class.edge_tracers`, `realm.enter_exit_and_private` |
| [#25](https://github.com/openharmony-rs/servo-quickjs/issues/25) | `JS_GetGCParameter`/`JS_SetGCParameter` do not persist, and `runtime.context_lifecycle` passes anyway | `runtime.context_lifecycle` |
| [#26](https://github.com/openharmony-rs/servo-quickjs/issues/26) | `RemoveRawValueRoot` retains a reference; `HasAnyMicroTasks` cannot see engine-queued reactions | `misc.roots_options_memory`, `promise.internal_job_queue` |
| [#28](https://github.com/openharmony-rs/servo-quickjs/issues/28) | `JS_FireOnNewGlobalObject` never dispatches | `debugger.new_global_object_hook` |

Each issue carries the full analysis, the code locations and the reproduction command; this file
only records which allowlist entry points at which issue.

## Entries that keep issue 57

`eval.environment_chain`, `edge.eval.compile_and_environment`, `edge.weakmap`, `edge.weakmap_overwrite`,
`objects.weakmap_wpt_get_entry` and `objects.weakmap_wpt_set_entry` stay on
<https://github.com/openharmony-rs/servo-ohos/issues/57>, which covers exactly those two defects: the
environment-chain append order and the holder-map key type.

## Differences that are not defects

`streaming.consume_callback` is not a defect at all. QuickJS has no WebAssembly, so it has no stream
consumer; the entry is `accepted` instead of `known-bug`.

`misc.standard_classes` is `accepted` as well: QuickJS defines every standard class eagerly, so a
global's resolve and enumerate hooks have no class left to add lazily. The contract case observes
different hook return values, but no JS-visible behaviour difference has been demonstrated.

`errors.core` is also `accepted`, but one of its steps shows a separate shim defect: QuickJS records
`JS_GetScriptedCallerPrivate` as `None` for a frame that has a private value, where SpiderMonkey
records `Some(77)`. That deserves its own entry once the step is split out of the compound case.
