# throwables

## Overview

The exceptions and errors that the injection subsystem throws: invalid
injector configuration, invalid injection points, invalid slices, group
validation failures, and unrecoverable injection errors.

## Interface

- `public class InjectionError extends MixinError`. Its constructors are `()`,
  `(String)`, `(Throwable)` and `(String, Throwable)`. Because it is an `Error`
  it is unchecked [InjectionError.java:InjectionError].
- `public class InjectionValidationException extends Exception` is checked. Its
  constructor is `(InjectorGroupInfo group, String message)` and it exposes
  `getGroup()` [InjectionValidationException.java:InjectionValidationException].
- `public class InvalidInjectionException extends InvalidMixinException` is
  unchecked. `InvalidMixinException` extends `MixinException`, which extends
  `RuntimeException` (see
  src/main/java/org/spongepowered/asm/mixin/transformer/throwables/InvalidMixinException.java).
  Constructors take an `IMixinContext` or an `ISelectorContext`, then a message
  and/or a `Throwable` cause, and optionally an `IActivityContext`. It exposes
  `ISelectorContext getContext()` [InvalidInjectionException.java:InvalidInjectionException].
- `public class InvalidInjectionPointException extends InvalidInjectionException`.
  Its constructors are `(IMixinContext | InjectionInfo, [Throwable cause,] String format, Object... args)`
  [InvalidInjectionPointException.java:InvalidInjectionPointException].
- `public class InvalidSliceException extends InvalidInjectionException`. Its
  constructors take an `IMixinContext` or an `ISliceContext`, then a message
  and/or a cause [InvalidSliceException.java:InvalidSliceException].

## Invariants

- `InvalidInjectionException.getContext()` returns the `ISelectorContext` given
  to the constructor. It returns `null` for every `IMixinContext` constructor.
  The `ISelectorContext` constructors pass `selectorContext.getMixin()` up as
  the mixin context [InvalidInjectionException.java:getContext].
- `InvalidInjectionPointException` builds its message with
  `String.format(format, args)`. The `InjectionInfo` overloads pass `info` to the
  `ISelectorContext` super constructor, so `getContext()` returns that `info`
  [InvalidInjectionPointException.java:InvalidInjectionPointException].
- `InvalidSliceException(ISliceContext owner, ...)` passes `owner.getMixin()` to
  the `IMixinContext` super constructor. As a result `getContext()` is `null`
  even when an `ISliceContext` was supplied [InvalidSliceException.java:InvalidSliceException].
- Serial version UIDs: `InvalidInjectionException` and
  `InvalidInjectionPointException` use `2L`. `InjectionError`,
  `InjectionValidationException` and `InvalidSliceException` use `1L`
  [InvalidInjectionException.java:serialVersionUID; InvalidInjectionPointException.java:serialVersionUID].

## Side Effects

- None

## Verification

- Compile: `mise exec -- ./gradlew compileJava` (run from the repository root).
- Test: `mise exec -- ./gradlew test`. `build.gradle` declares the `test`
  source set, but `src/test` does not exist yet, so there are no tests today.
