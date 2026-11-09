# org.spongepowered.asm.util.logging

## Overview

Package `org.spongepowered.asm.util.logging` holds `MessageRouter`, a static switch that sends `javax.annotation.processing.Messager` diagnostics either to the annotation processor's messager or, when none is installed, to Mixin's `"mixin"` logger. Code shared by the annotation processor (the `ap` source set) and the runtime reports messages through this switch.

## Interface

- `MessageRouter` is a final utility class with a private constructor [MessageRouter.java:class].
- `public static Messager getMessager()` never returns `null`. When the static field is `null`, it creates a `LoggingMessager`, caches it and returns it. Later calls return the same instance [MessageRouter.java:getMessager].
- `public static void setMessager(Messager)`:
  - `null` clears the field, so the next `getMessager()` creates a fresh `LoggingMessager`.
  - Any other value is stored as `new DebugInterceptingMessager(messager)`, so `getMessager()` returns the wrapper, not the argument.
  - Nothing prevents wrapping an already-wrapped messager a second time [MessageRouter.java:setMessager].
- `LoggingMessager` is package-private. All four `printMessage` overloads ignore their `Element`, `AnnotationMirror` and `AnnotationValue` arguments and call `logger.log(level, msg.toString())`. A `null` message throws `NullPointerException` [MessageRouter.java:LoggingMessager].
  - Kind-to-level mapping: `ERROR` becomes `Level.ERROR`; `WARNING` and `MANDATORY_WARNING` become `Level.WARN`; `NOTE` becomes `Level.INFO`; `OTHER` and anything else become `Level.DEBUG`. A `null` kind throws `NullPointerException` [MessageRouter.java:LoggingMessager.messageKindToLoggingLevel].
- `DebugInterceptingMessager(Messager)` is package-private. Each overload drops the call when `kind == Kind.OTHER`. Otherwise it forwards the identical arguments to the wrapped messager's overload of the same arity [MessageRouter.java:DebugInterceptingMessager].

## Invariants

- After `setMessager(non-null)`, the static `messager` field always holds a `DebugInterceptingMessager`. `Kind.OTHER` messages never reach the wrapped messager [MessageRouter.java:DebugInterceptingMessager].
- `LoggingMessager.logger` is `private static final`. It is obtained once, when the class initialises, from `MixinService.getService().getLogger("mixin")` [MessageRouter.java:LoggingMessager.logger].

## Side Effects

- `getMessager()` and `setMessager()` mutate a global static field with no synchronisation and no `volatile`. Tests should call `setMessager(null)` in teardown [MessageRouter.java:getMessager].
- The first `LoggingMessager` resolves `org.spongepowered.asm.service.MixinService`, so calling `getMessager()` with no messager installed needs a resolvable service [MessageRouter.java:LoggingMessager.logger].
- `LoggingMessager.printMessage` writes to the `org.spongepowered.asm.logging.ILogger` named `"mixin"` [MessageRouter.java:LoggingMessager.printMessage].

## Verification

Run these from the repository root:

- `mise exec -- ./gradlew compileJava`
- `mise exec -- ./gradlew test`

`build.gradle` declares a `test` source set, but `src/test` does not exist yet, so there are no tests today. `setMessager` with a mock `Messager` is testable without a `MixinService`: check that it wraps the messager, drops `Kind.OTHER` and forwards the other kinds.
