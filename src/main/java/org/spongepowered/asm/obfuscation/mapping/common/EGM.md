# org.spongepowered.asm.obfuscation.mapping.common

## Overview

This package holds the generic field and method mapping structs (`IMapping` implementations). Their javadoc says they store mapping information "during AP runs" [MappingField.java:MappingField] [MappingMethod.java:MappingMethod]. Most users are in the annotation processor under src/ap/java/org/spongepowered/tools/obfuscation. Inside `src/main`, the `mixin/injection/selectors` and `mixin/injection/struct` packages also use them. Both classes are immutable value objects. Every transform returns a new instance.

## Interface

- `public class MappingField implements IMapping<MappingField>` [MappingField.java:MappingField]:
  - constructors `(String owner, String name)` (desc is null) and `(String owner, String name, String desc)`
  - `getType()` returns `FIELD`
  - `getName()`, `final getSimpleName()`, `final getOwner()` and `final getDesc()`
  - `getSuper()` returns null
  - `move`, `remap`, `transform` and `copy`
  - `serialise()`, `equals`, `hashCode` and `toString`
- `public class MappingMethod implements IMapping<MappingMethod>` [MappingMethod.java:MappingMethod]:
  - constructors `(String fullyQualifiedName, String desc)` and `(String owner, String simpleName, String desc)`
  - `getType()` returns `METHOD`
  - `getName()`, `getSimpleName()`, `getOwner()` and `getDesc()`
  - `getSuper()` returns null
  - `boolean isConstructor()` and `MappingMethod addPrefix(String prefix)`
  - `move`, `remap`, `transform` and `copy`
  - `serialise()`, `equals`, `hashCode` and `toString`

## Invariants

### MappingField

- `toString()` and `serialise()` return `L<owner>;<name>:<desc>`. A null desc is written as an empty string, but a null owner or name appears as `null` [MappingField.java:toString].
- `equals` compares `toString()` values, and `hashCode` is `Objects.hashCode(toString())`. Owner, name and desc all take part [MappingField.java:equals] [MappingField.java:hashCode].
- `getName()` and `getSimpleName()` both return the bare name [MappingField.java:getName].

### MappingMethod

- The `(fullyQualifiedName, desc)` constructor splits on the last `/`. The owner is the part before it, or null when there is no `/`. The simple name is the part after it [MappingMethod.java:getOwnerFromName] [MappingMethod.java:getBaseName].
- `getName()` returns `<owner>/<name>`, or just `<name>` when the owner is null, or null when the name is null [MappingMethod.java:getName].
- `toString()` and `serialise()` return `getName()`, followed by `" " + desc` when desc is non-null [MappingMethod.java:toString].
- `equals` compares only the simple `name` and `desc`, not the owner. `hashCode` uses `getName()`, which includes the owner, plus `desc`. Two methods with the same name and desc but different owners are therefore equal yet usually have different hash codes [MappingMethod.java:equals] [MappingMethod.java:hashCode].
- `isConstructor()` is true when the name equals `<init>` [MappingMethod.java:isConstructor].
- `addPrefix(prefix)` returns `this` when the simple name is null or already starts with `prefix`. Otherwise it returns a new mapping named `prefix + name` [MappingMethod.java:addPrefix].

### Both classes

- `move`, `remap`, `transform` and `copy` return a new instance with the owner, name or desc replaced, and never change `this` [MappingField.java:move] [MappingMethod.java:move].

## Side Effects

- None. Every field is final and the package has no static state.

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileJava`
- Run from the repo root: `mise exec -- ./gradlew test`
- `build.gradle` declares a `test` source set, but `src/test` does not exist yet. `test` therefore runs no tests today.
