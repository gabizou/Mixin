# org.spongepowered.asm.mixin.gen.throwables

## Overview

This package holds `InvalidAccessorException`. Mixin throws it when an `@Accessor` or `@Invoker` method cannot be parsed, validated or matched to a target member. The code that throws it is in the parent package `mixin/gen` (`AccessorInfo` and `InvokerInfo`).

## Interface

- `public class InvalidAccessorException extends InvalidMixinException`, with `serialVersionUID = 2L` [InvalidAccessorException.java:InvalidAccessorException]. `InvalidMixinException` lives in src/main/java/org/spongepowered/asm/mixin/transformer/throwables and extends `MixinException`, which is a `RuntimeException`. The exception is therefore unchecked.
- Constructors that take an `IMixinContext`: `(IMixinContext, String)`, `(IMixinContext, Throwable)` and `(IMixinContext, String, Throwable)` [InvalidAccessorException.java:InvalidAccessorException]
- Constructors that take an `AccessorInfo`: `(AccessorInfo, String)`, `(AccessorInfo, Throwable)` and `(AccessorInfo, String, Throwable)` [InvalidAccessorException.java:InvalidAccessorException]
- `public AccessorInfo getAccessorInfo()` [InvalidAccessorException.java:getAccessorInfo]

## Invariants

- `getAccessorInfo()` returns null when the exception was built with an `IMixinContext` constructor. Otherwise it returns the `AccessorInfo` that was passed in [InvalidAccessorException.java:getAccessorInfo].
- The `AccessorInfo` constructors pass `info.getMixin()` to the superclass as the context. Passing a null `AccessorInfo` throws `NullPointerException` while the exception is being built [InvalidAccessorException.java:InvalidAccessorException].

## Side Effects

- None.

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileJava`
- Run from the repo root: `mise exec -- ./gradlew test`
- `build.gradle` declares a `test` source set, but `src/test` does not exist yet. `test` therefore runs no tests today.
