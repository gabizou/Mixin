# org.spongepowered.asm.mixin.throwables

## Overview

This package holds the base throwables for the mixin subsystem. There are two branches:

- Unchecked exceptions under `MixinException extends RuntimeException`, for invalid mixins or user state.
- Errors under `MixinError extends Error`, for faults in Mixin's own state and for failed required configs or mixins.

Many types outside this package extend these bases, for example `InvalidMixinException` in src/main/java/org/spongepowered/asm/mixin/transformer/throwables and `MixinInitialisationError` in src/main/java/org/spongepowered/asm/launch.

## Interface

- `public class MixinException extends RuntimeException`, with `serialVersionUID = 1L` [MixinException.java:MixinException]:
  - constructors `(String)`, `(String, IActivityContext)`, `(Throwable)`, `(Throwable, IActivityContext)`, `(String, Throwable)` and `(String, Throwable, IActivityContext)`
  - `public void prepend(IActivityContext upstreamContext)` [MixinException.java:prepend]
  - `public String getMessage()`, which is overridden [MixinException.java:getMessage]
- `public class ClassAlreadyLoadedException extends MixinException`, with constructors `(String)`, `(Throwable)` and `(String, Throwable)` [ClassAlreadyLoadedException.java:ClassAlreadyLoadedException]
- `public class ClassMetadataNotFoundException extends MixinException`. It has the same six constructors as `MixinException`, including the `IActivityContext` variants [ClassMetadataNotFoundException.java:ClassMetadataNotFoundException]
- `public class MixinError extends Error`, with constructors `()`, `(String)`, `(Throwable)` and `(String, Throwable)` [MixinError.java:MixinError]
- `public class MixinApplyError extends MixinError`, with constructors `(String)`, `(Throwable)` and `(String, Throwable)` [MixinApplyError.java:MixinApplyError]
- `public class MixinPrepareError extends MixinError`, with constructors `(String)`, `(Throwable)` and `(String, Throwable)` [MixinPrepareError.java:MixinPrepareError]

## Invariants

- Every class here declares `serialVersionUID = 1L` [MixinException.java:serialVersionUID] [MixinError.java:serialVersionUID].
- The `IActivityContext` constructors store `context.toString()` as `activityDescriptor`. A null context leaves it null [MixinException.java:MixinException].
- `getMessage()` returns `super.getMessage()` when `activityDescriptor` is null, and `super.getMessage() + " [" + activityDescriptor + "]"` otherwise [MixinException.java:getMessage].
- `prepend(upstream)` sets `activityDescriptor` to `upstream + " -> " + activityDescriptor`, or to `" -> " + upstream` when it was null. The glue string is `ActivityStack.GLUE_STRING`, which is `" -> "`, in src/main/java/org/spongepowered/asm/mixin/transformer/ActivityStack.java. A null `upstreamContext` throws `NullPointerException` [MixinException.java:prepend].
- `MixinApplyError` and `MixinPrepareError` are the errors for required mixins and required configs. Their javadoc says the first is thrown when applying a required mixin fails and the second when initialising a required config fails [MixinApplyError.java:MixinApplyError] [MixinPrepareError.java:MixinPrepareError].
- The javadoc of `ClassMetadataNotFoundException` says it is the unchecked stand-in for `ClassNotFoundException`, thrown when `ClassInfo.forName` returns null and the caller needs a failure [ClassMetadataNotFoundException.java:ClassMetadataNotFoundException].

## Side Effects

- None. `prepend` changes only the exception's own `activityDescriptor` field [MixinException.java:prepend].

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileJava`
- Run from the repo root: `mise exec -- ./gradlew test`
- `build.gradle` declares a `test` source set, but `src/test` does not exist yet. `test` therefore runs no tests today.
