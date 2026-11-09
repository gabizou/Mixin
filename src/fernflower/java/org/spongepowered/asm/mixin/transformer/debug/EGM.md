# org.spongepowered.asm.mixin.transformer.debug (fernflower source set)

## Overview

This package is the runtime decompiler for Mixin's debug class export. It uses
the Fernflower decompiler to turn each exported `.class` file into a `.java`
source file. `RuntimeDecompiler` decompiles synchronously on the caller's
thread, and `RuntimeDecompilerAsync` queues files for a background thread. Both
implement `IDecompiler`, defined in
src/main/java/org/spongepowered/asm/mixin/transformer/ext/IDecompiler.java,
which has one method, `void decompile(File file)`.

Nothing references these classes at compile time.
src/main/java/org/spongepowered/asm/mixin/transformer/ext/extensions/ExtensionClassExporter.java
loads them by name in `initDecompiler`:

1. If the option `DEBUG_EXPORT_DECOMPILE` (property
   `mixin.debug.export.decompile`) is off, it returns `null`.
2. It loads `org.spongepowered.asm.mixin.transformer.debug.RuntimeDecompiler`.
   If `DEBUG_EXPORT_DECOMPILE_THREADED` (property
   `mixin.debug.export.decompile.async`) is on, it appends `Async` to that name.
3. It calls the class's `(File)` constructor with
   `new File(Constants.DEBUG_OUTPUT_DIR, "java")`, which is `.mixin.out/java`
   relative to the working directory.
4. If any `Throwable` is thrown, for example because Fernflower is not on the
   classpath, it logs at INFO "Fernflower could not be loaded, exported classes
   will not be decompiled. {}: {}" and leaves the decompiler `null`.

`ExtensionClassExporter.export` writes each class to
`.mixin.out/class/<name>.class` (`dumpClass`). If a decompiler is present, it
then calls `decompile(outputFile)`.

## Interface

### RuntimeDecompiler

- `public class RuntimeDecompiler extends IFernflowerLogger implements IDecompiler, IResultSaver`
  [RuntimeDecompiler.java:RuntimeDecompiler]
- `public RuntimeDecompiler(File outputPath)`
  [RuntimeDecompiler.java:RuntimeDecompiler(File)]
- `public void decompile(File file)` [RuntimeDecompiler.java:decompile]
- `public void saveClassFile(String path, String qualifiedName, String entryName, String content, int[] mapping)`
  writes the decompiled source [RuntimeDecompiler.java:saveClassFile].
- `public String toString()` returns the file name of the jar or directory that
  `Fernflower.class` was loaded from. If any `Exception` is thrown, it returns
  `"unknown source (classpath)"` [RuntimeDecompiler.java:toString].
  `ExtensionClassExporter` logs this value.
- `protected final ILogger logger` is
  `MixinService.getService().getLogger("fernflower")`
  [RuntimeDecompiler.java:logger].
- It overrides these `IFernflowerLogger` methods:
  - `startReadingClass(String)`
  - `writeMessage(String, Severity)`
  - `writeMessage(String, Throwable)`
  - `writeMessage(String, Severity, Throwable)`
- Seven `IResultSaver` methods have empty bodies:
  - `saveFolder`
  - `copyFile`
  - `createArchive`
  - `saveDirEntry`
  - `copyEntry`
  - `saveClassEntry`
  - `closeArchive`

  See [RuntimeDecompiler.java:saveFolder] and
  [RuntimeDecompiler.java:closeArchive].
- The Fernflower options are a `private final Map<String, Object> options`,
  built as an `ImmutableMap` [RuntimeDecompiler.java:options]:

| `IFernflowerPreferences` key | Value |
| --- | --- |
| `DECOMPILE_INNER` | `"0"` |
| `REMOVE_BRIDGE` | `"0"` |
| `DECOMPILE_GENERIC_SIGNATURES` | `"1"` |
| `ASCII_STRING_CHARACTERS` | `"1"` |
| `DECOMPILE_ENUM` | `"1"` |
| `HIDE_DEFAULT_CONSTRUCTOR` | `"1"` |
| `INDENT_STRING` | `"    "` (four spaces) |

### RuntimeDecompilerAsync

- `public class RuntimeDecompilerAsync extends RuntimeDecompiler implements Runnable, UncaughtExceptionHandler`
  [RuntimeDecompilerAsync.java:RuntimeDecompilerAsync]
- `public RuntimeDecompilerAsync(File outputPath)`
  [RuntimeDecompilerAsync.java:RuntimeDecompilerAsync(File)]
- `public void decompile(File file)` [RuntimeDecompilerAsync.java:decompile]
- `public void run()`, the worker loop [RuntimeDecompilerAsync.java:run]
- `public void uncaughtException(Thread thread, Throwable ex)`
  [RuntimeDecompilerAsync.java:uncaughtException]

## Invariants

- The constructor does not validate `outputPath` and declares no checked
  exceptions. A `null` path throws `NullPointerException` at
  `outputPath.exists()` [RuntimeDecompiler.java:RuntimeDecompiler(File)].
- Each `decompile` call builds a new
  `Fernflower(bytecodeProvider, this, options, this)`. The same object is both
  the result saver and the logger [RuntimeDecompiler.java:decompile].
- The anonymous `IBytecodeProvider` reads `new File(externalPath)` once and
  caches the bytes. Every later `getBytecode` call on that instance returns the
  same bytes, whatever path is requested. It ignores `internalPath`
  [RuntimeDecompiler.java:decompile].
- `decompile` first calls Fernflower's `addSource(File)` by reflection. On a
  `ReflectiveOperationException` it falls back to the older API,
  `getStructContext().addSpace(file, true)`. It then calls `decompileContext()`
  [RuntimeDecompiler.java:decompile].
- `decompile` never throws. It catches any `Throwable` and logs at WARN
  "Decompilation error while processing {}" with `file.getName()`. The
  exception itself is not logged [RuntimeDecompiler.java:decompile].
- `saveClassFile` uses only `qualifiedName` and `content`. It ignores `path`,
  `entryName` and `mapping` [RuntimeDecompiler.java:saveClassFile].
- Logging of Fernflower messages
  [RuntimeDecompiler.java:SEVERITY_LEVELS] [RuntimeDecompiler.java:writeMessage]:
  - Fernflower severities map by ordinal to `{TRACE, INFO, WARN, ERROR}`.
  - `writeMessage(String, Severity, Throwable)` passes the throwable to the
    logger only when the severity is `Severity.ERROR`.
  - `writeMessage(String, Throwable)` logs at WARN "{} {}: {}" with the message,
    the throwable's simple class name and its message. No stack trace is
    logged.
- How `RuntimeDecompilerAsync.decompile` behaves depends on the `run` flag
  [RuntimeDecompilerAsync.java:decompile]:
  - While `run` is `true`, it only adds the file to an unbounded
    `LinkedBlockingQueue<File>`.
  - Once `run` is `false`, it decompiles synchronously on the calling thread.
  - `run` is a plain field, not `volatile`.
- The worker loop takes a file from the queue and calls `super.decompile`
  [RuntimeDecompilerAsync.java:run]:
  - On `InterruptedException` it sets `run = false` and exits. Files still in
    the queue are not decompiled.
  - Any other `Exception` is printed with `printStackTrace()`, and the loop
    continues.
- After an uncaught throwable, `uncaughtException` logs at ERROR and then calls
  `flush()` [RuntimeDecompilerAsync.java:uncaughtException]. `flush()` sets
  `run = false` and drains the queue. It decompiles each file synchronously on
  the thread that runs the handler.
- There is no method to stop the worker or to wait for the queue to empty.
  Because the thread is a daemon, files still queued when the JVM exits are not
  decompiled.

## Side Effects

- The constructor deletes `outputPath` recursively if it exists and is a
  directory, using `org.spongepowered.asm.util.Files.deleteRecursively`
  (src/main/java/org/spongepowered/asm/util/Files.java)
  [RuntimeDecompiler.java:RuntimeDecompiler(File)]:
  - If `outputPath` is a regular file, nothing is deleted.
  - An `IOException` from the delete is logged at DEBUG as "Error cleaning
    output directory: {}" and otherwise ignored.
  - The constructor creates no directories.
- `decompile` reads the input `.class` file from disk with
  `InterpreterUtil.getBytes(new File(externalPath))`
  [RuntimeDecompiler.java:decompile].
- `saveClassFile` writes the source file [RuntimeDecompiler.java:saveClassFile]:
  - The target is `new File(outputPath, qualifiedName + ".java")`, written as
    UTF-8 with Guava `Files.write(content, file, Charsets.UTF_8)`.
  - Before writing, it calls `getParentFile().mkdirs()` and logs at INFO
    "Writing {}" with the absolute path.
  - If the write throws `IOException`, it logs at WARN
    "Cannot write source file <file> <ExceptionSimpleName>: <message>" and
    continues.
- `startReadingClass` logs at INFO "Decompiling {}"
  [RuntimeDecompiler.java:startReadingClass].
- All logging goes to the `fernflower` logger obtained from `MixinService`
  [RuntimeDecompiler.java:logger].
- The `RuntimeDecompilerAsync` constructor starts a background thread named
  `"Decompiler thread"`. The thread is a daemon at `Thread.MIN_PRIORITY`, with
  the decompiler itself as its uncaught-exception handler
  [RuntimeDecompilerAsync.java:RuntimeDecompilerAsync(File)].
- `uncaughtException` logs at ERROR "Async decompiler encountered an error and
  will terminate. Further decompile requests will be handled synchronously. {}
  {}" with the exception's class name and message
  [RuntimeDecompilerAsync.java:uncaughtException].

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileFernflowerJava`.
- The `fernflower` source set is declared at build.gradle:99-104. It compiles
  against `main.output` with `languageVersion = 8`, `compatibility = '1.6'` and
  `modularityExcluded = true`. That flag keeps its output out of the
  `org.spongepowered.mixin` module build (build.gradle:324-329).
- Its dependencies:
  - `fernflowerImplementation` extends `implementation` (build.gradle:75).
  - It adds `org.jetbrains.java.decompiler:fernflower:sponge-SNAPSHOT`
    (build.gradle:178).
  - That artifact is resolved from the `sponge` Maven repository at
    https://repo.spongepowered.org/maven (build.gradle:59-63).
- No tests cover this package at this commit.
