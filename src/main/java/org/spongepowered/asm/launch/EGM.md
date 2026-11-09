# org.spongepowered.asm.launch

## Overview

This package is the environment-agnostic entry point for starting the Mixin subsystem:

- `MixinBootstrap` runs the guarded `start` / `doInit` / `inject` sequence, records the active version in the global property store, and owns the `MixinPlatformManager` singleton.
- `GlobalProperties` is a static facade over the service-provided `IGlobalPropertyService` (the "blackboard").
- `MixinInitialisationError` reports a version clash between two Mixin copies.

The phase methods `start`, `doInit` and `inject` are package-private. Launcher integrations in other source sets declare classes in the same `org.spongepowered.asm.launch` package so they can call them:

| Launcher | Class (repo path) | Call |
| --- | --- | --- |
| LaunchWrapper | src/launchwrapper/java/org/spongepowered/asm/launch/MixinTweaker.java | the constructor calls `start()`, `acceptOptions` calls `doInit(CommandLineOptions.ofArgs(args))`, `injectIntoClassLoader` calls `inject()`, and `getLaunchTarget` calls `getPlatform().getLaunchTarget()` |
| LaunchWrapper | src/launchwrapper/java/org/spongepowered/asm/mixin/EnvironmentStateTweaker.java | `MixinBootstrap.getPlatform().inject()` |
| ModLauncher | src/modlauncher/java/org/spongepowered/asm/launch/MixinTransformationServiceAbstract.java | `initialize(IEnvironment)` calls `start()` |
| ModLauncher | src/modlauncher/java/org/spongepowered/asm/launch/MixinLaunchPluginLegacy.java | `doInit(CommandLineOptions.of(commandLineMixins))` and then `inject()` |

## Interface

- [platform](./platform/EGM.md): `MixinPlatformManager`, `MixinContainer`, platform agents, `MainAttributes`, `CommandLineOptions`, `MixinConnectorManager` and the `container` subpackage.
- `public abstract class MixinBootstrap`, which has a private constructor [MixinBootstrap.java:MixinBootstrap]:
  - `public static final String VERSION = "0.8.7"` [MixinBootstrap.java:VERSION]
  - `public static void init()` [MixinBootstrap.java:init]
  - `public static MixinPlatformManager getPlatform()` [MixinBootstrap.java:getPlatform]
  - `@Deprecated public static void addProxy()`, which only calls `MixinService.getService().beginPhase()` [MixinBootstrap.java:addProxy]
  - package-private `static boolean start()`, `static void doInit(CommandLineOptions)`, `@Deprecated static void doInit(List<String>)` (which delegates through `CommandLineOptions.ofArgs`) and `static void inject()` [MixinBootstrap.java:start] [MixinBootstrap.java:doInit] [MixinBootstrap.java:inject]
- `public final class GlobalProperties` [GlobalProperties.java:GlobalProperties]:
  - `static <T> T get(Keys)` and `static <T> T get(Keys, T defaultValue)`
  - `static void put(Keys, Object)`
  - `static String getString(Keys, String defaultValue)`
- `public static final class GlobalProperties.Keys`, with `public static Keys of(String name)` and `toString()`, which returns the name [GlobalProperties.java:Keys]. It defines these constants:
  - `INIT = "mixin.initialised"`
  - `AGENTS = "mixin.agents"`
  - `CONFIGS = "mixin.configs"`
  - `PLATFORM_MANAGER = "mixin.platform"`
  - `FML_LOAD_CORE_MOD = "mixin.launch.fml.loadcoremodmethod"`
  - `FML_GET_REPARSEABLE_COREMODS = "mixin.launch.fml.reparseablecoremodsmethod"`
  - `FML_CORE_MOD_MANAGER = "mixin.launch.fml.coremodmanagerclass"`
  - `FML_GET_IGNORED_MODS = "mixin.launch.fml.ignoredmodsmethod"`
- `public class MixinInitialisationError extends MixinError`, with constructors `()`, `(String)`, `(Throwable)` and `(String, Throwable)` [MixinInitialisationError.java:MixinInitialisationError]

## Invariants

### Bootstrap order

1. Class initialization of `MixinBootstrap` runs `MixinService.boot()`, then `MixinService.getService().prepare()`, then `getLogger("mixin")` [MixinBootstrap.java:static initializer].
2. `start()` first checks whether the `INIT` global is already set [MixinBootstrap.java:start] [MixinBootstrap.java:checkSubsystemVersion]:
   - If it is set to the same version, `start()` returns false and does nothing else.
   - If it is set to a different version, `start()` throws `MixinInitialisationError("Mixin subsystem version <active> was already initialised. Cannot bootstrap version 0.8.7")`.
3. Otherwise `start()` writes `INIT = VERSION` and offers the internals. If the static `initialised` flag is false, it then sets the flag, reads `getInitialPhase()` from the service, and continues by phase [MixinBootstrap.java:start]:
   - For `Phase.DEFAULT`: it logs an error, calls `MixinEnvironment.init(DEFAULT)` and `getPlatform().prepare(CommandLineOptions.defaultArgs())`, and sets `initState = false`.
   - For any other phase: it calls `MixinEnvironment.init(phase)`.
   - Both paths then call `service.beginPhase()`.
4. Finally `start()` calls `getPlatform()` and returns true [MixinBootstrap.java:start].
5. `doInit(args)` before `start()` behaves as follows [MixinBootstrap.java:doInit]:
   - If `INIT` is set, which happens when another Mixin copy started, it logs the warning `Multiple Mixin containers present, init suppressed for 0.8.7` and returns.
   - Otherwise it throws `IllegalStateException("MixinBootstrap.doInit() called before MixinBootstrap.start()")`.
6. `doInit(args)` after `start()` calls `getPlatform().getPhaseProviderClasses()` and discards the result. If `initState` is true, it then calls `getPlatform().prepare(args)` and `MixinService.getService().init()`. Repeated `doInit` calls repeat that work, because there is no guard [MixinBootstrap.java:doInit].
7. `inject()` only delegates to `getPlatform().inject()`. The guard against repeated calls is in `MixinPlatformManager` [MixinBootstrap.java:inject].
8. `init()` is `start()` followed by `doInit(CommandLineOptions.defaultArgs())`. If `start()` returns false, `doInit` is skipped [MixinBootstrap.java:init].

### Platform manager and internals

- `getPlatform()` caches the manager in the static `platform` field. If the `PLATFORM_MANAGER` global already holds a `MixinPlatformManager`, for example one created by another copy, it reuses it without calling `init()`. Otherwise it creates a new manager, stores it under `PLATFORM_MANAGER` and then calls `init()` [MixinBootstrap.java:getPlatform].
- `offerInternals` loads `org.spongepowered.asm.mixin.transformer.MixinTransformer$Factory` by reflection, calling `setAccessible(true)` on its no-argument constructor, and passes the instance to `service.offer(...)`. A `ReflectiveOperationException` becomes `MixinError`, which propagates out of `start()` after `INIT` has already been written. An `AbstractMethodError` is printed and swallowed [MixinBootstrap.java:getInternals] [MixinBootstrap.java:offerInternals].

### GlobalProperties

- `Keys.of(name)` returns the same instance for equal names. The registry is the static, unsynchronized `HashMap` `keys`, which is created lazily [GlobalProperties.java:Keys.of].
- `Keys.resolve(service)` caches the first non-null `service.resolveKey(name)` in the key instance. It returns null when the service is null and nothing is cached yet [GlobalProperties.java:Keys.resolve].
- `GlobalProperties.service` is fetched once from `MixinService.getGlobalPropertyService()` and never reset [GlobalProperties.java:getService].
- None of the accessors check the service for null [GlobalProperties.java:get].
- The `get`, `put` and `getString` semantics, including when defaults apply, are defined by the `IGlobalPropertyService` implementation that `MixinService` selects. See the sibling package src/main/java/org/spongepowered/asm/service.

## Side Effects

- Static state that nothing resets [MixinBootstrap.java:initialised] [MixinBootstrap.java:platform] [GlobalProperties.java:service] [GlobalProperties.java:Keys.keys]:
  - `MixinBootstrap.initialised` (initially false)
  - `MixinBootstrap.initState` (initially true)
  - `MixinBootstrap.logger` and `MixinBootstrap.platform`
  - `GlobalProperties.service` and `GlobalProperties.Keys.keys`
- Global property writes: `INIT` is set to `"0.8.7"` [MixinBootstrap.java:registerSubsystem], and `PLATFORM_MANAGER` is set to the new manager [MixinBootstrap.java:getPlatform].
- Service calls: `prepare()` and `getLogger` from the static initializer, `offer`, `getInitialPhase` and `beginPhase` from `start()`, and `init()` from `doInit` [MixinBootstrap.java:start] [MixinBootstrap.java:doInit].
- `MixinEnvironment.init(phase)` initializes global environment state in src/main/java/org/spongepowered/asm/mixin/MixinEnvironment.java [MixinBootstrap.java:start].
- Logging and console output: `Initialising mixin subsystem after game pre-init phase! Some mixins may be skipped.` at error [MixinBootstrap.java:start], `Multiple Mixin containers present, init suppressed for {}` at warn [MixinBootstrap.java:doInit], and `printStackTrace()` for `AbstractMethodError` [MixinBootstrap.java:offerInternals].

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileJava`
- Run from the repo root: `mise exec -- ./gradlew test`
- `build.gradle` declares a `test` source set, but `src/test` does not exist yet. `test` therefore runs no tests today.
