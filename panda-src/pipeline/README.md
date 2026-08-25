# Pipeline — Threading, Locking, and Pipelined-Rendering Primitives

**Source:** `panda/src/pipeline/` · Library: `libp3pipeline` · Notify categories: `pipeline`, `thread`

`pipeline` is Panda3D's low-level concurrency layer: mutexes, condition
variables, threads, and the "pipeline cycler" system that lets the app,
cull, and draw stages of the render pipeline run on different threads while
each reads a *different, consistent* copy of scene-graph state. Nearly
everything above this module (scene graph nodes, `RenderState`, `GeomVertexData`,
...) stores its mutable data in a `CycleData` subclass guarded by a
`PipelineCycler`, rather than locking a plain mutex per object — see "Core
concepts" below for why.

This directory documents the public C++ API of every class in
`panda/src/pipeline`, for use without re-reading the engine source.

## Class map

```
Locking primitives (all have a Debug/Direct or Debug/Simple-etc. build-time impl — see Core concepts)
  Mutex                  (Mutex.md)          — non-reentrant lock
  ReMutex                (ReMutex.md)        — reentrant lock
  LightMutex              (LightMutex.md)     — non-reentrant, no ConditionVar support, no-op under SIMPLE_THREADS
  LightReMutex             (LightReMutex.md)   — reentrant + lightweight
  ConditionVar            (ConditionVar.md)   — wait()/notify(), one waiter woken
  ConditionVarFull        (ConditionVarFull.md) — wait()/notify()/notify_all()
  Semaphore               (Semaphore.md)      — counting semaphore, built on Mutex + ConditionVar
  MutexHolder / LightMutexHolder / ReMutexHolder / LightReMutexHolder
                          (MutexHolder.md)    — RAII acquire/release wrappers
  CyclerHolder             (CyclerHolder.md)   — RAII wrapper for PipelineCyclerBase

Threading
  Thread                  (Thread.md)         — abstract base; subclass + override thread_main()
  ├── MainThread            (MainThread.md)     — singleton, the main thread
  ├── ExternalThread         (ExternalThread.md) — singleton, "some other non-Panda thread"
  ├── GenericThread          (GenericThread.md)  — spawn from a plain C function pointer
  └── PythonThread            (PythonThread.md)   — spawn from a Python callable (HAVE_PYTHON only)
  ThreadSimpleManager      (ThreadSimpleManager.md) — cooperative-thread scheduler (THREAD_SIMPLE_IMPL only)

Pipelining / double-buffering
  Pipeline                (Pipeline.md)       — owns and cycles a collection of PipelineCyclers
  PipelineCycler<T>        (PipelineCycler.md) — per-pipeline-stage copies of a CycleData page
  CycleData                (CycleData.md)      — base class for one page of cycled data
  CycleDataReader<T>       (CycleDataReader.md)       — fast unlocked read
  CycleDataLockedReader<T> (CycleDataLockedReader.md) — locked read, can elevate to a writer
  CycleDataWriter<T>       (CycleDataWriter.md)       — read-write access
```

`ThreadPriority` (an enum, not a class) and the impl-selection classes for
each locking primitive and `Thread` (`MutexDebug`/`MutexDirect`/
`MutexSimpleImpl`/..., `ThreadPosixImpl`/`ThreadWin32Impl`/..., `BlockerSimple`,
the `contextSwitch.h`/`.c` register-save primitives) are documented as
**"Implementation variants"** subsections inside the doc of the public class
they back, not as standalone files — see "Core concepts" below for why.
`config_pipeline.{h,cxx}` (library init + config vars, covered below),
`p3pipeline_composite{1,2}.cxx` (unity-build wrapper files), and the
`test_*.cxx` standalone test programs are not documented as classes.

## Core concepts

**Every locking/threading primitive has multiple build-time implementations,
and exactly one is compiled in.** `Mutex`, `ReMutex`, `LightMutex`,
`LightReMutex`, `ConditionVar`, `ConditionVarFull`, and `Thread` are all
public-facing classes that inherit from (or typedef to) whichever impl class
is selected by preprocessor macros at build time — `DEBUG_THREADS`,
`THREAD_SIMPLE_IMPL`, `THREAD_POSIX_IMPL`, `THREAD_WIN32_IMPL`,
`THREAD_DUMMY_IMPL`, `MUTEX_SPINLOCK`. You never construct or reference an
impl class directly; you use the public class and the right impl is already
baked in. Each class's own doc has an "Implementation variants" section
listing which impl backs it under which macro, and any variant-specific
caveat.

**`DEBUG_THREADS` trades speed for deadlock detection.** When defined, every
`Mutex`/`ReMutex`/`ConditionVar`/`ConditionVarFull` is backed by a `*Debug`
impl (`MutexDebug`, `ConditionVarDebug`, ...) that tracks which thread holds
which lock and, before blocking, walks the "who is thread X waiting on, and
who holds *that* lock" chain to detect cycles synchronously and report them.
Release builds use the `*Direct` impls instead, which forward straight to the
platform primitive with no bookkeeping. This is a compile-time choice, not a
runtime toggle.

**Three threading models, chosen at build time:** true OS threads
(`THREAD_POSIX_IMPL`/`THREAD_WIN32_IMPL` — real parallelism), cooperative
"simple" threads (`THREAD_SIMPLE_IMPL` — single-CPU, setjmp/longjmp-based
context switching managed by [`ThreadSimpleManager`](ThreadSimpleManager.md)),
or no threading at all (`THREAD_DUMMY_IMPL` — `Thread::start()` always
fails). Under `SIMPLE_THREADS`, `LightMutex`/`LightReMutex` compile to
**no-ops** (a cooperative switch can't happen mid-critical-section, so no
locking is needed) — don't use `ConditionVar`/`ConditionVarFull` with a
`LightMutex` for this reason, they need a real lock. Code written to run
under `SIMPLE_THREADS` must call `Thread::consider_yield()` periodically, or
it starves every other thread — there's no preemption.

**`PipelineCycler` exists so the app/cull/draw stages can each see a
different, consistent snapshot without heavyweight per-object locking.**
[`Pipeline`](Pipeline.md) (usually the single `get_render_pipeline()`
instance) maintains N "stages," one per pipeline stage in flight. Any class
whose data needs to be safely readable by one stage while another stage
writes it (e.g. `RenderState`, `TransformState`, `GeomVertexData`) stores
that data in a [`CycleData`](CycleData.md) subclass and owns a
[`PipelineCycler<T>`](PipelineCycler.md); [`Pipeline::cycle()`](Pipeline.md)
advances every registered cycler to the next stage once per frame,
copy-on-writing only the cyclers that were actually written to. Read/write
access always goes through a [`CycleDataReader`](CycleDataReader.md),
[`CycleDataLockedReader`](CycleDataLockedReader.md), or
[`CycleDataWriter`](CycleDataWriter.md) — never the cycler directly.

**Cycling deliberately skips-and-retries rather than blocking, to avoid one
cycler deadlocking another.** `Pipeline::cycle()` walks its list of "dirty"
cyclers; if a cycler's lock can't be acquired immediately, `cycle()` leaves
it for the next pass instead of blocking on it — except when it's the very
last cycler left to process, in which case it *does* block, which is what
lets `DEBUG_THREADS`' deadlock detector catch a genuine deadlock rather than
`cycle()` spinning forever.

**Without `DO_PIPELINING` defined, cycling degenerates to near-zero cost.**
`PipelineCyclerBase` selects `PipelineCyclerTrueImpl` (real multi-stage
cycling with locking) when `DO_PIPELINING` is defined, `PipelineCyclerDummyImpl`
(single-stage, adds sanity checks like read/write-balance assertions, no
locking) in single-threaded dev builds, or `PipelineCyclerTrivialImpl`
(a single raw `CycleData*`, no checks at all) in the minimum-overhead case.

**The `CycleData*Reader`/`*Writer` template classes have no Python
bindings by design.** All six (`CycleDataReader`, `CycleDataLockedReader`,
`CycleDataStageReader`, `CycleDataLockedStageReader`, `CycleDataWriter`,
`CycleDataStageWriter`) are wrapped in `#ifndef CPPPARSER`, hidden from
`interrogate` "to improve compile-time speed and memory utilization" — they're
a C++-only RAII convenience layer.

**Config variables** (`config_pipeline.h`, `init_libpipeline()`):
`support-threads` (bool, whether threading is enabled at all — even if the
platform supports it), `thread-stack-size` (stack size for spawned threads),
and `name-deleted-mutexes` (dev-only debug aid that keeps a destroyed mutex's
name around so a stale reference to it reports something meaningful instead
of garbage — explicitly documented to leak memory, never enable in
production). Notify categories: `pipeline` (the module in general) and
`thread` (thread lifecycle specifically).

## File index

| Class | Purpose |
|---|---|
| [Mutex.md](Mutex.md) | Non-reentrant mutual-exclusion lock |
| [ReMutex.md](ReMutex.md) | Reentrant mutex — same thread may lock repeatedly |
| [LightMutex.md](LightMutex.md) | Lightweight non-reentrant mutex, no-op under SIMPLE_THREADS |
| [LightReMutex.md](LightReMutex.md) | Lightweight reentrant mutex |
| [ConditionVar.md](ConditionVar.md) | Condition variable, `wait()`/`notify()` only |
| [ConditionVarFull.md](ConditionVarFull.md) | Condition variable with `notify_all()` |
| [Semaphore.md](Semaphore.md) | Counting semaphore |
| [MutexHolder.md](MutexHolder.md) | RAII acquire/release wrapper family (Mutex/LightMutex/ReMutex/LightReMutex) |
| [CyclerHolder.md](CyclerHolder.md) | RAII acquire/release wrapper for `PipelineCyclerBase` |
| [Thread.md](Thread.md) | Abstract base for a lightweight process/thread |
| [MainThread.md](MainThread.md) | Singleton representing the main thread |
| [ExternalThread.md](ExternalThread.md) | Singleton representing a non-Panda-spawned thread |
| [GenericThread.md](GenericThread.md) | Spawn a thread from a plain function pointer |
| [PythonThread.md](PythonThread.md) | Spawn a thread running a Python callable |
| [ThreadSimpleManager.md](ThreadSimpleManager.md) | Scheduler for cooperative `SIMPLE_THREADS` |
| [Pipeline.md](Pipeline.md) | Owns and cycles a collection of `PipelineCycler`s |
| [PipelineCycler.md](PipelineCycler.md) | Per-pipeline-stage copies of a `CycleData` page |
| [CycleData.md](CycleData.md) | Base class for one page of cycled data |
| [CycleDataReader.md](CycleDataReader.md) | Fast unlocked read access |
| [CycleDataLockedReader.md](CycleDataLockedReader.md) | Locked read access, elevatable to write |
| [CycleDataWriter.md](CycleDataWriter.md) | Read-write access |

## Status

pipeline — done (2026-08-25). Other `panda/src/*` subsystems not yet
documented — see `../../README.md` for the overall index.
