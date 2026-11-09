# org.spongepowered.asm.logging

## Overview

This package is Mixin's logging abstraction. `ILogger` is a subset of the Log4j2 `Logger` API. Its javadoc says it exists so Mixin core does not depend on log4j2 and so logging can be delegated to the mixin service [ILogger.java:ILogger]. Services return adapters from `IMixinService.getLogger(name)`.

Where each adapter is used:

- `MixinServiceAbstract.createLogger` returns `LoggerAdapterDefault`, which discards everything.
- `MixinService` uses `LoggerAdapterConsole` with a stderr debug stream, but only when starting the service throws an `Error`.
- The launcher services override `createLogger` to return their own `LoggerAdapterLog4j2`. See src/launchwrapper/java/org/spongepowered/asm/service/mojang/MixinServiceLaunchWrapper.java and src/modlauncher/java/org/spongepowered/asm/service/modlauncher/MixinServiceModLauncher.java.
- Nothing under `src/` references `LoggerAdapterJava`.

## Interface

- `public interface ILogger` [ILogger.java:ILogger]:
  - `String getId()` and `String getType()`
  - `void catching(Level, Throwable)` and `void catching(Throwable)`
  - `void log(Level, String, Object...)` and `void log(Level, String, Throwable)`
  - `<T extends Throwable> T throwing(T)`
  - For each of `trace`, `debug`, `info`, `warn`, `error` and `fatal`: `(String message, Object... params)` and `(String message, Throwable t)`
- `public enum Level { FATAL, ERROR, WARN, INFO, DEBUG, TRACE }` [Level.java:Level]
- `public abstract class LoggerAdapterAbstract implements ILogger`, with constructor `protected LoggerAdapterAbstract(String id)` and `getId()` [LoggerAdapterAbstract.java:LoggerAdapterAbstract]
- `public static class LoggerAdapterAbstract.FormattedMessage` [LoggerAdapterAbstract.java:FormattedMessage]:
  - `FormattedMessage(String message, Object... params)`
  - `String getMessage()` and `toString()`
  - `boolean hasThrowable()` and `Throwable getThrowable()`
- `public class LoggerAdapterConsole extends LoggerAdapterAbstract`, with `LoggerAdapterConsole(String name)` and the fluent `LoggerAdapterConsole setDebugStream(PrintStream)` [LoggerAdapterConsole.java:LoggerAdapterConsole]
- `public class LoggerAdapterDefault extends LoggerAdapterAbstract`, with `LoggerAdapterDefault(String name)` [LoggerAdapterDefault.java:LoggerAdapterDefault]
- `public class LoggerAdapterJava extends LoggerAdapterAbstract`, with `LoggerAdapterJava(String name)` [LoggerAdapterJava.java:LoggerAdapterJava]

## Invariants

### Levels and message formatting

- `Level` declares its constants in this order: `FATAL`, `ERROR`, `WARN`, `INFO`, `DEBUG`, `TRACE`. `LoggerAdapterJava` indexes its level table by `ordinal()`, so reordering the enum changes the mapping [Level.java:Level] [LoggerAdapterJava.java:LEVELS].
- `FormattedMessage` substitutes each `{}` with the next parameter, left to right, and keeps any text after the last substitution [LoggerAdapterAbstract.java:FormattedMessage]:
  - Escapes are not supported.
  - With no parameters, the message is returned as-is.
  - Placeholders left over when the parameters run out are kept literally.
- The last parameter is stored as the throwable only when it is a `Throwable` and was not used by a placeholder. For example, `("a {}", t)` puts `t` into the text, and `("a {}: {}", x, y, t)` stores `t` [LoggerAdapterAbstract.java:FormattedMessage].
- `LoggerAdapterAbstract` routes each level method to `log(Level, ...)`, and `catching(Throwable)` calls `catching(Level.WARN, t)` [LoggerAdapterAbstract.java:catching].

### LoggerAdapterConsole

- Output streams: `FATAL`, `ERROR` and `WARN` go to `System.err`, and `INFO` goes to `System.out`. `DEBUG` goes to the debug stream, which is null by default, so debug output is dropped. `TRACE` is always dropped [LoggerAdapterConsole.java:getOutputStream].
- A line has the format `[HH:mm:ss] [<id>/<LEVEL>] <message>`. Any throwable is printed to the same stream. `log(Level, String, Throwable)` does not substitute placeholders [LoggerAdapterConsole.java:log].
- A null name becomes `""` [LoggerAdapterConsole.java:LoggerAdapterConsole].
- `catching(level, t)` ignores `level` and logs `Catching <class>: <message>` at `WARN`. `throwing(t)` logs `Throwing <class>: <message>` at `WARN` and returns `t` [LoggerAdapterConsole.java:catching] [LoggerAdapterConsole.java:throwing].

### LoggerAdapterDefault

- `getType()` returns `"Default Logger (No Logging)"`. `log` and `catching` do nothing. `throwing(t)` returns null, not `t` [LoggerAdapterDefault.java:throwing].

### LoggerAdapterJava

- Level mapping: `FATAL` and `ERROR` become `SEVERE`, `WARN` becomes `WARNING`, `INFO` stays `INFO`, `DEBUG` becomes `FINE` and `TRACE` becomes `FINER` [LoggerAdapterJava.java:LEVELS].
- A throwable is logged as a second record containing `t.toString()`, without a stack trace [LoggerAdapterJava.java:log].
- The constructor uses `LogManager.getLogManager().getLogger(name)`. That call returns only an already-registered logger, and the adapter falls back to the global logger (`Logger.GLOBAL_LOGGER_NAME`) when none exists. It never creates a named logger [LoggerAdapterJava.java:getLogger].
- `catching(level, t)` ignores `level` and warns. `throwing(t)` warns and returns `t` [LoggerAdapterJava.java:catching] [LoggerAdapterJava.java:throwing].

## Side Effects

- `LoggerAdapterConsole` writes to `System.out` and `System.err`, and to the debug `PrintStream` once one is set. `setDebugStream` changes the instance [LoggerAdapterConsole.java:log] [LoggerAdapterConsole.java:setDebugStream].
- `LoggerAdapterConsole.DATE_FORMAT` is one static `SimpleDateFormat` shared by all instances. `SimpleDateFormat` is not thread-safe, and access to it is not synchronized [LoggerAdapterConsole.java:DATE_FORMAT].
- `LoggerAdapterJava` writes to `java.util.logging` loggers [LoggerAdapterJava.java:log].
- `LoggerAdapterDefault` has no side effects.

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileJava`
- Run from the repo root: `mise exec -- ./gradlew test`
- `build.gradle` declares a `test` source set, but `src/test` does not exist yet. `test` therefore runs no tests today.
