# transformer/ext/extensions

## Overview

This package holds the three built-in `IExtension` implementations. The
`Extensions` registry in the parent `ext` package runs them around each target
class's mixin application. Their registration order is set by
`DefaultExtensions` in the `transformer` package.

- `ExtensionCheckClass` runs ASM's `CheckClassAdapter` over the transformed class.
- `ExtensionCheckInterfaces` writes an audit report of interface methods that the target does not implement.
- `ExtensionClassExporter` dumps transformed bytecode to disk. It can also pass the dumped file to a Fernflower decompiler, which it loads by reflection.

## Interface

- `ExtensionCheckClass` (public, implicit no-arg constructor). `postApply(ITargetClassContext)` does the work. `preApply` and `export` are no-ops [ExtensionCheckClass.java:ExtensionCheckClass]
- `ExtensionCheckClass.ValidationFailedException extends MixinException` (unchecked). Constructors: `(String, Throwable)`, `(String)`, `(Throwable)` [ExtensionCheckClass.java:ValidationFailedException]
- `ExtensionCheckInterfaces()` (public). `preApply` and `postApply` do the work. `export` is a no-op [ExtensionCheckInterfaces.java:ExtensionCheckInterfaces]
- `ExtensionClassExporter(MixinEnvironment env)` (public). Methods:
  - `export(MixinEnvironment, String name, boolean force, ClassNode)`
  - `File dumpClass(String fileName, ClassNode)`, which appends `.class` to `fileName`
  - `boolean isDecompilerActive()`

  `preApply` and `postApply` are no-ops [ExtensionClassExporter.java:ExtensionClassExporter]

## Invariants

- Activation (`checkActive`):
  - `ExtensionCheckClass` is active only when `Option.DEBUG_VERIFY` is set [ExtensionCheckClass.java:checkActive]
  - `ExtensionCheckInterfaces` is active only when `Option.CHECK_IMPLEMENTS` is set. As a side effect, `checkActive` refreshes `strict` from `Option.CHECK_IMPLEMENTS_STRICT` [ExtensionCheckInterfaces.java:checkActive]
  - `ExtensionClassExporter.checkActive` always returns `true` [ExtensionClassExporter.java:checkActive]
- `ExtensionCheckClass.postApply` rethrows any `RuntimeException` from `CheckClassAdapter` as `ValidationFailedException(ex.getMessage(), ex)`. `Error`s are not wrapped [ExtensionCheckClass.java:postApply]
- `ExtensionCheckInterfaces`:
  - `preApply` records the target's pre-mixin interface methods, from `getInterfaceMethods(false)` [ExtensionCheckInterfaces.java:preApply]
  - `postApply` skips abstract targets unless `strict` is set. On that early return, the entries recorded by `preApply` are not removed [ExtensionCheckInterfaces.java:postApply]
  - `postApply` reports a method as missing only if two conditions hold. First, `findMethodInHierarchy(name, desc, ALL_CLASSES, Traversal.ALL)` returns null or an abstract method. Second, the method is not in the superclass's interface methods or the pre-mixin set. `getSuperClass()` is not null-checked [ExtensionCheckInterfaces.java:postApply]
- `ExtensionCheckInterfaces.start()` runs once per instance (`started` flag). The constructor computes paths only and does no I/O [ExtensionCheckInterfaces.java:start]
- `ExtensionClassExporter.export` writes only when `force` is true or `Option.DEBUG_EXPORT` is set. Without `force`, the name must also match `Option.DEBUG_EXPORT_FILTER`, if one is set [ExtensionClassExporter.java:export]
- The export filter is a case-insensitive, full-string match. `**` matches any sequence, `*` matches one or more characters other than `.`, and `?` matches exactly one character. Every other character is a literal [ExtensionClassExporter.java:prepareFilter] [ExtensionClassExporter.java:applyFilter]
- Decompiler setup only happens when `Option.DEBUG_EXPORT_DECOMPILE` is set. The decompiler class is `org.spongepowered.asm.mixin.transformer.debug.RuntimeDecompiler`, with an `Async` suffix when `Option.DEBUG_EXPORT_DECOMPILE_THREADED` is set. Any `Throwable` during loading is swallowed, which leaves `isDecompilerActive() == false` [ExtensionClassExporter.java:initDecompiler]
- `dumpClass` always returns `new File(<.mixin.out/class>, fileName + ".class")`, even if serialisation fails. `IOException` is swallowed [ExtensionClassExporter.java:dumpClass]
- `getClassBytes` first tries `COMPUTE_FRAMES`. On `NegativeArraySizeException` it retries once with flags `0`. Any other failure returns `null` [ExtensionClassExporter.java:getClassBytes]

## Side Effects

- Class initialisation of `ExtensionCheckInterfaces` and `ExtensionClassExporter` calls `MixinService.getService().getLogger("mixin")`, so a `MixinService` must be available [ExtensionCheckInterfaces.java:logger] [ExtensionClassExporter.java:logger]
- The `ExtensionClassExporter` constructor recursively deletes `.mixin.out/class` (`Constants.DEBUG_OUTPUT_DIR`). A failure is logged at DEBUG [ExtensionClassExporter.java:ExtensionClassExporter]
- `dumpClass` runs `mkdirs` on the parent directories and writes `<name>.class` under `.mixin.out/class`. The decompiler gets `.mixin.out/java` as its output directory [ExtensionClassExporter.java:dumpClass] [ExtensionClassExporter.java:ExtensionClassExporter]
- `initDecompiler` calls `Class.forName` reflectively and logs INFO for the attempt, the success and the failure [ExtensionClassExporter.java:initDecompiler]
- `ExtensionCheckInterfaces.start()` creates `.mixin.out/audit`. It overwrites `mixin_implementation_report.csv` (header `Class,Method,Signature,Interface`) and `mixin_implementation_report.txt`, both in ISO-8859-1 [ExtensionCheckInterfaces.java:start]
- For each missing method, `postApply` appends a CSV row. When the missing count is above 0, it prints a `PrettyPrinter` table to `System.err` and appends the same table to the TXT report. It logs INFO when it skips an abstract target [ExtensionCheckInterfaces.java:postApply] [ExtensionCheckInterfaces.java:appendToCSVReport]
- `export` uses the `"export"` profiler. The section is not ended on exception [ExtensionClassExporter.java:export]

## Verification

Run from the repository root:

- `mise exec -- ./gradlew compileJava`
- `mise exec -- ./gradlew test`

The `test` source set is declared in `build.gradle`, but `src/test` does not exist yet, so there are no tests today.
