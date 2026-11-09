# org.spongepowered.asm.util.perf

## Overview

Package `org.spongepowered.asm.util.perf` holds `Profiler`, Mixin's lightweight wall-clock profiler. It keeps a stack of named sections per profiler instance, splits recorded time into named phases, and renders the results as a `PrettyPrinter` table or summary. A process-wide registry and a global on/off flag decide whether new sections record time.

## Interface

### Static API

- `public static final int ROOT = 0x01` and `public static final int FINE = 0x02` are the flags for `begin(int, ...)` [Profiler.java:ROOT] [Profiler.java:FINE].
- `static void setActive(boolean)` writes the global `active` flag. There is no getter. The flag fixes the concrete section type when a new name is first seen: `LiveSection` when active, `DisabledSection` when not. It also enables `SubSection` creation and the `IllegalStateException` thrown from `end` [Profiler.java:setActive].
- `static Profiler getProfiler(String id)` returns the registered instance for `id`, creating and registering it on first use. The same id always returns the same instance [Profiler.java:getProfiler].
- `static Collection<Profiler> getProfilers()` returns an immutable snapshot (`ImmutableList`) in `HashMap` order [Profiler.java:getProfilers].
- `static void printAuditSummary()` merges every registered profiler into `ResultSection` aggregates and prints a summary. Its id is the profiler ids joined with `,`, and its phases are the deduplicated union in first-seen order. Missing required sections are auto-created as empty, so this path never throws `NullPointerException` [Profiler.java:printAuditSummary].

### Instance API

- `Profiler(String id)` does not register the new instance (only `getProfiler` does). `phases` starts as `["Initial"]`. `toString()` returns the id [Profiler.java:Profiler(String)].
- `Section get(String name)` returns the section for that exact key, creating it if needed. It does not push the section or start timing [Profiler.java:get].
- `begin` overloads:
  - `begin(String...)` and `begin(int flags, String...)` join the path with `.`, so `("a","b","c")` becomes `"a.b.c"` [Profiler.java:begin(int,String...)].
  - `begin(int flags, String name)` builds a key from the current head: `head + "." + name`, or with `ROOT`, the key `name` plus, when active, a `SubSection` keyed `head + " -> " + name` whose delegate is the root section.
  - It sets the fine and root flags on the pushed section, pushes it, starts it and returns it. It throws nothing itself [Profiler.java:begin(int,String)].
- `void mark(String phase)` sums `getTime()` over all sections. When the sum is `0` ms, it renames the current (last) phase to `phase`. Otherwise it appends `phase` and calls `mark()` on every section, which moves the current time into that section's per-phase history [Profiler.java:mark].
- `void reset()` invalidates and clears all sections, resets phases to `["Initial"]` and clears the stack. The registry and the `active` flag are untouched [Profiler.java:reset].
- `Collection<Section> getSections()` returns a live, unmodifiable view in `TreeMap` key order [Profiler.java:getSections].
- `PrettyPrinter printer(boolean includeFine, boolean group)` builds the table without printing it [Profiler.java:printer(boolean,boolean)]:
  - Its columns are `Section`, `TOTAL`, one column per phase, `Count` and `Avg.`.
  - Times print as `"<n> ms"`, and the average uses `DecimalFormat("   ###0.000 ms")` with the default locale. Empty cells print `-` [Profiler.java:printSectionRow].
- `void printSummary()` prints a summary to `System.err`. It throws `NullPointerException` unless all of these section keys exist: `mixin.prepare`, `mixin.read`, `mixin.apply`, `mixin.write`, `mixin`, `class.load`, `class.transform`, `mixin.debug.export` [Profiler.java:printSummary()].
  - Percentages divide by the `mixin` total, so a zero total produces `NaN` rather than an exception [Profiler.java:printSummary(String,List,Map)].

### `Section` hierarchy

- `Section` is a public abstract static class whose constructor is package-private [Profiler.java:Section].
  - Its public API is `getName()`, `getBaseName()`, `setInfo`/`getInfo()` (the info defaults to the name), `isRoot()`, `isFine()`, `end()`, `next(String)`, `getTime()`, `getTotalTime()`, `getSeconds()`, `getTotalSeconds()`, `getTimes()`, `getCount()`, `getTotalCount()`, `getAverageTime()`, `getTotalAverageTime()`, and a final `toString()` that returns the name.
  - The base implementation returns zeros, and its `getTimes()` returns `{0}`.
- `DisabledSection` records nothing. `end()` pops it from the profiler stack. `next(name)` ends it and returns `begin(name)` with flags `0`, so `ROOT` and `FINE` are not carried over [Profiler.java:DisabledSection].
- `LiveSection` times with `System.currentTimeMillis()` [Profiler.java:LiveSection]:
  - `stop()` adds the elapsed time only while running, but it always increments `count`, so ending a section twice increments `count` without adding time.
  - `getTime()` counts completed periods in the current phase only. `getTotalTime()` is `time + markedTime`.
  - `getTimes()` returns `long[cursor+1]`, whose last element is the current-phase time.
  - Averages return `0.0` when the current `count == 0`, even after earlier phases recorded counts.
- `SubSection` forwards `start`, `stop` and `info` to its root delegate. `getBaseName()` returns the parent head name and `getDelegate()` returns the root. While active, `next(name)` ends the root delegate, which is normally not on the stack, so it can drain the stack and throw `IllegalStateException` [Profiler.java:SubSection].
- `ResultSection` sums its added sections. It is used only by `printAuditSummary` [Profiler.java:ResultSection].

## Invariants

- `phases` is never empty [Profiler.java:reset].
- Each profiler has exactly one section per key. Its type (`LiveSection` or `DisabledSection`) is fixed by `active` when the section is created [Profiler.java:get].
- `begin` always pushes exactly one section and returns that started instance [Profiler.java:begin(int,String)].
- Stack pops compare by identity. Ending a section pops entries until that section is found. If the stack empties first, `end` throws `IllegalStateException("Attempted to pop <name> but the stack is empty")` when active and stays silent when inactive. Entries already popped stay removed [Profiler.java:end(Section)].
- An invalidated section (after `reset()`) never pops the stack again, but a `LiveSection` still updates its own time and count [Profiler.java:reset].
- Thread safety: `reset`, `get`, `begin`, `end`, `mark` and `getSections` synchronise on the instance. `printer`, `printSummary()` and the section counters do not. `active` is a plain static field, neither `volatile` nor synchronised [Profiler.java:Profiler].

## Side Effects

- `private static final Map<String, Profiler> profilers` is a global registry that is never cleared. It has no remove API, so entries persist across tests in the same JVM [Profiler.java:profilers].
- `private static boolean active` is global. Tests must set and restore it [Profiler.java:active].
- `printSummary()` and `printAuditSummary()` write to `System.err` through `PrettyPrinter.print()` [Profiler.java:printSummary(String,List,Map)].
- The summary path calls `MixinService.getService().getClassProvider().findAgentClass("org.spongepowered.metronome.Agent", false)` and reflectively invokes its static `getTimes()`. Every `Throwable` from that block is swallowed [Profiler.java:printSummary(String,List,Map)].

## Verification

Run these from the repository root:

- `mise exec -- ./gradlew compileJava`
- `mise exec -- ./gradlew test`

`build.gradle` declares a `test` source set, but `src/test` does not exist yet, so there are no tests today. For deterministic unit tests, use `new Profiler(id)` rather than `getProfiler` (which writes to the global registry) and restore `setActive(...)` in teardown. Assert on counts, keys, phases and `IllegalStateException` behaviour rather than on millisecond timings.
