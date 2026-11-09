# selectors/throwables

## Overview

Unchecked exceptions for target selector failures. `SelectorException` keeps
the `ITargetSelector` that failed. `SelectorConstraintException` reports that a
selector matched too few or too many targets. For example, `BeforeInvoke`
throws it when a selector matches fewer than its minimum count (see
src/main/java/org/spongepowered/asm/mixin/injection/points/BeforeInvoke.java).

## Interface

- `public class SelectorException extends MixinException`. `MixinException`
  extends `RuntimeException` (see
  src/main/java/org/spongepowered/asm/mixin/throwables/MixinException.java)
  [SelectorException.java:SelectorException].
  - Constructors: `(ITargetSelector, String)`, `(ITargetSelector, Throwable)`
    and `(ITargetSelector, String, Throwable)`
    [SelectorException.java:SelectorException(ITargetSelector,String)].
  - `ITargetSelector getSelector()` [SelectorException.java:getSelector].
- `public class SelectorConstraintException extends SelectorException` has the
  same three constructor shapes [SelectorConstraintException.java:SelectorConstraintException].

## Invariants

- `selector` is a `private final` field. Every constructor sets it without a
  null check, so `getSelector()` returns exactly the selector passed in, even
  `null` [SelectorException.java:selector].
- Both classes declare `serialVersionUID = 1L`
  [SelectorException.java:serialVersionUID; SelectorConstraintException.java:serialVersionUID].
- Neither class exposes the `IActivityContext` constructor variants that
  `MixinException` declares [SelectorException.java:SelectorException].

## Side Effects

- None

## Verification

- Compile: `mise exec -- ./gradlew compileJava` (run from the repository root).
- Test: `mise exec -- ./gradlew test`. `build.gradle` declares the `test`
  source set, but `src/test` does not exist yet, so there are no tests today.
