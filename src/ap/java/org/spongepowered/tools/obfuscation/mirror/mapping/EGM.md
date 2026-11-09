# mirror/mapping

## Overview

Holds `MappingMethodResolvable`, a `MappingMethod` that keeps a `TypeHandle`
for its owner. The handle lets mapping lookups move up the class hierarchy
that the annotation processor resolved at compile time. `TypeHandle` and
`MethodHandle` in the parent `mirror` package create these objects. The
method lookup loop in
`src/ap/java/org/spongepowered/tools/obfuscation/ObfuscationEnvironment.java`
calls `getSuper()` on them.

## Interface

- `MappingMethodResolvable`: public final class that extends
  `org.spongepowered.asm.obfuscation.mapping.common.MappingMethod`. The
  constructor is `(TypeHandle owner, String name, String desc)`
  [MappingMethodResolvable.java:MappingMethodResolvable].
- `getSuper()`: returns the same method on the nearest supertype that declares
  it [MappingMethodResolvable.java:getSuper].
- `move(TypeHandle newOwner)`: copies the method onto a new owner handle
  [MappingMethodResolvable.java:move].
- `remap(String newName)`, `transform(String newDesc)` and `copy()`: copy the
  method and keep the owner handle [MappingMethodResolvable.java:copy].

## Invariants

- If the owner handle is `null`, `getSuper()` defers to
  `MappingMethod.getSuper()` [MappingMethodResolvable.java:getSuper].
- Otherwise `getSuper()` converts the descriptor to a Java signature with
  `TypeUtils.getJavaSignature(desc)` and checks the hierarchy in this order:
  1. If the direct superclass has a matching method (`findMethod(name, signature)`),
     return `superClass.getMappingMethod(name, desc)`.
  2. Otherwise, return the same from the first directly implemented interface
     that has a matching method.
  3. Otherwise, if a superclass exists, return
     `superClass.getMappingMethod(name, desc).getSuper()`.
  4. Otherwise, return `MappingMethod.getSuper()`.
  [MappingMethodResolvable.java:getSuper]
- `remap`, `transform` and `copy` return new `MappingMethodResolvable`
  objects. They never return a plain `MappingMethod`
  [MappingMethodResolvable.java:remap].

## Side Effects

- None. `getSuper()` only queries `TypeHandle` objects, which can resolve more
  type handles through their `ITypeHandleProvider` [MappingMethodResolvable.java:getSuper].

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileApJava`
- Run: `mise exec -- ./gradlew test`. Note that `src/test` does not exist yet,
  so no tests cover this package today.
