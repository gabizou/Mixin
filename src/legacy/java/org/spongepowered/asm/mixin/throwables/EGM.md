# org.spongepowered.asm.mixin.throwables (legacy source set)

## Overview

This directory holds the legacy source set's part of the
`org.spongepowered.asm.mixin.throwables` package. The package is split across
source sets. The other throwables (`MixinError`, `MixinApplyError` and others)
are in src/main/java/org/spongepowered/asm/mixin/throwables. The one class here,
`CompanionPluginError`, is the error raised when a companion plugin uses an
unsupported legacy API. Its Javadoc is "Thrown for unsupported legacy API
calls".

## Interface

- `public class CompanionPluginError extends LinkageError`
  [CompanionPluginError.java:CompanionPluginError].
- `private static final long serialVersionUID = 1L`
  [CompanionPluginError.java:serialVersionUID].
- Constructors:
  - `public CompanionPluginError()`, with an empty body
    [CompanionPluginError.java:CompanionPluginError()].
  - `public CompanionPluginError(String message)`, which calls `super(message)`
    [CompanionPluginError.java:CompanionPluginError(String)].
  - `public CompanionPluginError(String message, Throwable cause)`, which calls
    `super(message, cause)`
    [CompanionPluginError.java:CompanionPluginError(String,Throwable)].

## Invariants

- `CompanionPluginError` is a `java.lang.Error`, not an `Exception`, because it
  extends `LinkageError`. A `catch (Exception)` block does not catch it
  [CompanionPluginError.java:CompanionPluginError].
- The file has no imports. It uses only `java.lang` types
  [CompanionPluginError.java:import].
- The legacy shims throw it only through the single-argument constructor:
  - message `"ClassNode.<method>"` from
    src/legacy/java/org/spongepowered/asm/lib/tree/ClassNode.java;
  - message `"ClassVisitor.<method>"` from
    src/legacy/java/org/spongepowered/asm/lib/ClassVisitor.java.
- Outside this partition,
  src/main/java/org/spongepowered/asm/mixin/transformer/PluginHandle.java
  (`applyLegacy`) uses the `(String, Throwable)` constructor. It wraps
  `LinkageError`, `IllegalAccessException`, `IllegalArgumentException` and the
  cause of an `InvocationTargetException` raised by a legacy plugin call.

## Side Effects

- None.

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileLegacyJava`.
- The `legacy` source set is declared at build.gradle:85-88 with
  `compatibility = '1.6'`.
- build.gradle:220 excludes `**/throwables` from Javadoc.
- No tests cover this package at this commit.
