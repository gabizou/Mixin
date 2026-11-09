# transformer/throwables

## Overview

This package holds the transformer's throwable types, and all of them are
unchecked. The Exception branch extends
`org.spongepowered.asm.mixin.throwables.MixinException`, which extends
`RuntimeException`. The Error branch extends
`org.spongepowered.asm.mixin.throwables.MixinError`, which extends
`java.lang.Error`. No class in this package overrides `getMessage`.
`MixinException.getMessage` appends `" [" + activity + "]"` when an
`IActivityContext` is supplied.

## Interface

Error branch, which `catch (Exception)` does not catch:

- `MixinTransformerError extends MixinError`. Constructors: `(String)`, `(Throwable)`, `(String, Throwable)` [MixinTransformerError.java:MixinTransformerError]
- `IllegalClassLoadError extends MixinTransformerError`, with the same constructors [IllegalClassLoadError.java:IllegalClassLoadError]
- `ReEntrantTransformerError extends MixinTransformerError`, with the same constructors [ReEntrantTransformerError.java:ReEntrantTransformerError]

Exception branch:

- `InvalidMixinException extends MixinException`. It has 12 constructors. Each takes `IMixinInfo` or `IMixinContext`, plus a `String` message, a `Throwable` cause, or both, plus an optional `IActivityContext`. `IMixinInfo getMixin()` [InvalidMixinException.java:InvalidMixinException] [InvalidMixinException.java:getMixin]
- `InvalidInterfaceMixinException extends InvalidMixinException`. It has 6 constructors, none of which takes an `IActivityContext` [InvalidInterfaceMixinException.java:InvalidInterfaceMixinException]
- `MixinApplicatorException extends InvalidMixinException`. It has 12 constructors [MixinApplicatorException.java:MixinApplicatorException]
- `MixinTargetAlreadyLoadedException extends InvalidMixinException`. Constructors: `(IMixinInfo, String message, String target)`, `(IMixinInfo, String, String, Throwable)`. `String getTarget()` [MixinTargetAlreadyLoadedException.java:getTarget]
- `MixinPreProcessorException extends MixinException`. Constructors: `(String, IActivityContext)`, `(String, Throwable, IActivityContext)`. It has no mixin accessor [MixinPreProcessorException.java:MixinPreProcessorException]
- `MixinReloadException extends MixinException`. Constructor: `(IMixinInfo, String)`. `IMixinInfo getMixinInfo()`. It is not an `InvalidMixinException` [MixinReloadException.java:getMixinInfo]

## Invariants

- The `IMixinContext` constructors of `InvalidMixinException` store `context.getMixin()`. A `null` context throws `NullPointerException` [InvalidMixinException.java:InvalidMixinException]
- `getMixin()`, `getTarget()` and `getMixinInfo()` return the constructor arguments unchanged [InvalidMixinException.java:getMixin] [MixinTargetAlreadyLoadedException.java:getTarget] [MixinReloadException.java:getMixinInfo]
- `MixinApplicatorException` constructors without an activity context pass `(ActivityStack) null`, so `getMessage()` is the raw message [MixinApplicatorException.java:MixinApplicatorException]
- None of these classes defines `getContext`, `getMessage` or `toString` [InvalidMixinException.java:InvalidMixinException]

## Side Effects

- None. These are plain exception types.

## Verification

Run from the repository root:

- `mise exec -- ./gradlew compileJava`
- `mise exec -- ./gradlew test`

The `test` source set is declared in `build.gradle`, but `src/test` does not exist yet, so there are no tests today.
