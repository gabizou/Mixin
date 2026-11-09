# org.spongepowered.asm.obfuscation.mapping

## Overview

This package defines `IMapping<TMapping>`, the common interface for member mapping structs (field, method, class and package). It works like a self-typed builder: every transform returns a copy of the implementing type. The implementations live in the subpackages listed below. Most callers are in the annotation processor under src/ap/java/org/spongepowered/tools/obfuscation.

## Interface

- `public interface IMapping<TMapping>` [IMapping.java:IMapping]:
  - `Type getType()`
  - `TMapping move(String newOwner)`, `TMapping remap(String newName)`, `TMapping transform(String newDesc)` and `TMapping copy()`
  - `String getName()`, `String getSimpleName()`, `String getOwner()` and `String getDesc()`
  - `TMapping getSuper()` and `String serialise()`
- `public enum IMapping.Type { FIELD, METHOD, CLASS, PACKAGE }` [IMapping.java:Type]
- [common](./common/EGM.md): `MappingField` and `MappingMethod`, the generic implementations.
- [mcp](./mcp/EGM.md): `MappingFieldSrg`, a `MappingField` that serialises to its SRG string.
- [fg3](./fg3/EGM.md): contains only `package-info.java` in `src/main`.

## Invariants

- The javadoc contract for `move`, `remap`, `transform` and `copy` is to "create a clone" with the given part replaced. Implementations must not change the receiver [IMapping.java:move] [IMapping.java:copy].
- `getName()` includes the owner for method mappings, while `getSimpleName()` returns the bare member name [IMapping.java:getName] [IMapping.java:getSimpleName].
- `getOwner()` and `getDesc()` may return null. The javadoc says owner is undefined for packages and desc is undefined for classes and packages [IMapping.java:getOwner] [IMapping.java:getDesc].
- `getSuper()` may return null when no overridden member exists or it cannot be resolved [IMapping.java:getSuper].
- `serialise()` is documented as a convenience, and writers may use their own format [IMapping.java:serialise].

## Side Effects

- None. The package contains only an interface and an enum.

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileJava`
- Run from the repo root: `mise exec -- ./gradlew test`
- `build.gradle` declares a `test` source set, but `src/test` does not exist yet. `test` therefore runs no tests today.
