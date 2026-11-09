# transformer/ext

## Overview

The transformer extension SPI. The interfaces here define the extension hooks
that run around each target class, the class-generator chain, the decompiler,
the hot-swap agent, and the target-class view. `Extensions` is the single
concrete registry. It stores extensions and class generators, selects the
active extensions for each `MixinEnvironment`, and sends `preApply`,
`postApply` and `export` to them in order. This package contains no
implementations of `IDecompiler` or `IHotSwap`.

## Interface

- `IExtension`: `checkActive(MixinEnvironment)`, `preApply(ITargetClassContext)`, `postApply(ITargetClassContext)`, `export(MixinEnvironment, String name, boolean force, ClassNode)` [IExtension.java:IExtension]
- `IExtensionRegistry`: `getExtensions()`, `getActiveExtensions()`, `<T extends IExtension> T getExtension(Class)`, `getSyntheticClassRegistry()` [IExtensionRegistry.java:IExtensionRegistry]
- `IClassGenerator`: `String getName()`, `boolean generate(String name, ClassNode)` [IClassGenerator.java:IClassGenerator]
- `IDecompiler`: `void decompile(File)` [IDecompiler.java:IDecompiler]
- `IHotSwap`: `registerMixinClass(String)`, `registerTargetClass(String, ClassNode)` [IHotSwap.java:IHotSwap]
- `ITargetClassContext`: `ClassInfo getClassInfo()`, `ClassNode getClassNode()` [ITargetClassContext.java:ITargetClassContext]
- `final class Extensions implements IExtensionRegistry`:
  - Constructor `Extensions(ISyntheticClassRegistry)`
  - Registration: `add(IExtension)`, `add(IClassGenerator)`
  - Activation and dispatch: `select(MixinEnvironment)`, `preApply`, `postApply`, `export`
  - Generator lookup: `getGenerators()`, `<T extends IClassGenerator> T getGenerator(Class)`

  [Extensions.java:Extensions]
- Child package: [extensions](./extensions/EGM.md) contains the built-in `ExtensionCheckClass`, `ExtensionCheckInterfaces` and `ExtensionClassExporter`.

## Invariants

- Before the first `select()`, `getActiveExtensions()` returns an empty list. Until then `preApply`, `postApply` and `export` do nothing [Extensions.java:activeExtensions]
- Each call to `select(env)` rebuilds the active list as an `ImmutableList`. The list holds the registered extensions whose `checkActive(env)` returns true, in registration order. An extension added later is inactive until the next `select` [Extensions.java:select]
- `preApply`, `postApply` and `export` iterate only the active extensions, in registration order, and catch nothing. The first exception aborts the loop. `export` passes `force` through unchanged [Extensions.java:preApply] [Extensions.java:postApply] [Extensions.java:export]
- `add(...)` appends to the list, which allows duplicates, and keys the instance by `getClass()`. A second instance of the same class overwrites the map entry, but both stay in the list [Extensions.java:add]
- How `getExtension` and `getGenerator` look things up:
  - They try an exact class-key hit first.
  - Otherwise they return the first registered element that is assignable to the requested type, and cache it under that type.
  - If nothing matches, they throw `IllegalArgumentException("Extension for <...> could not be found")`. Generator lookups use the same message. This contradicts the `IExtensionRegistry` javadoc, which promises `null`.

  [Extensions.java:lookup] [IExtensionRegistry.java:getExtension]
- A lookup cached under a supertype key is never invalidated by a later `add` [Extensions.java:lookup]
- `getExtensions()` returns a new unmodifiable view on every call. `getGenerators()` returns the same unmodifiable view every time, with all generators and no environment filtering [Extensions.java:getExtensions] [Extensions.java:getGenerators]

## Side Effects

- On a cache miss, `getExtension` and `getGenerator` write to the instance map [Extensions.java:lookup]
- All state belongs to the `Extensions` instance. This package does no file I/O, no logging and no class loading.

## Verification

Run from the repository root:

- `mise exec -- ./gradlew compileJava`
- `mise exec -- ./gradlew test`

The `test` source set is declared in `build.gradle`, but `src/test` does not exist yet, so there are no tests today.
