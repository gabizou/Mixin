# org.spongepowered.asm.obfuscation

## Overview

This package holds obfuscation structs shared by the runtime and the annotation processor. At the top level there is one class, `RemapperChain`. It is an `IRemapper` that passes each lookup through every registered `IRemapper` in order. `MixinEnvironment` holds one instance per environment in a final field and exposes it through `getRemappers()`, and `MixinTargetContext` uses it to remap references. Both classes are in src/main/java/org/spongepowered/asm/mixin and src/main/java/org/spongepowered/asm/mixin/transformer.

## Interface

- `public class RemapperChain implements IRemapper` [RemapperChain.java:RemapperChain]. `IRemapper` is in src/main/java/org/spongepowered/asm/mixin/extensibility.
  - `public RemapperChain add(IRemapper remapper)` is fluent [RemapperChain.java:add]
  - `mapMethodName(owner, name, desc)`, `mapFieldName(owner, name, desc)`, `map(typeName)`, `unmap(typeName)`, `mapDesc(desc)` and `unmapDesc(desc)` [RemapperChain.java:map]
  - `toString()` returns `RemapperChain[<count>]` [RemapperChain.java:toString]
- [mapping](./mapping/EGM.md): the `IMapping` interface and its `common`, `mcp` and `fg3` subpackages.

## Invariants

- Remappers run in the order they were added, and each one receives the output of the one before. A remapper that returns null, or a value equal to its input, leaves the current value unchanged [RemapperChain.java:mapMethodName] [RemapperChain.java:map] [RemapperChain.java:mapDesc].
- For `mapMethodName` and `mapFieldName`, only `name` moves along the chain. `owner` and `desc` are passed to every remapper unchanged [RemapperChain.java:mapMethodName] [RemapperChain.java:mapFieldName].
- An empty chain returns its input unchanged [RemapperChain.java:map].
- `add` accepts duplicates and null, and does not check either. A null remapper causes `NullPointerException` on the next lookup [RemapperChain.java:add].

## Side Effects

- `add` appends to the instance's `ArrayList`, which is not synchronized [RemapperChain.java:remappers].
- The package has no static state.

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileJava`
- Run from the repo root: `mise exec -- ./gradlew test`
- `build.gradle` declares a `test` source set, but `src/test` does not exist yet. `test` therefore runs no tests today.
