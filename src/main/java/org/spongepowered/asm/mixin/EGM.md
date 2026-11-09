# org.spongepowered.asm.mixin

## Overview

This is the public consumer API of Mixin. It holds the annotations that mixin authors put on mixin classes (`@Mixin`, `@Shadow`, `@Overwrite` and others), the `Mixins` registry for configuration files, and `MixinEnvironment`. `MixinEnvironment` holds the global runtime state: the phase, side, options, compatibility level and active transformer. The subpackages implement injection, transformation, accessor generation and refmaps.

## Interface

### Annotations

All annotation elements and defaults below are exactly as written in the source.

| Annotation | `@Target` | Retention | Elements |
| --- | --- | --- | --- |
| `@Mixin` [Mixin.java:Mixin] | TYPE | CLASS | `Class<?>[] value() default {}`, `String[] targets() default {}`, `int priority() default 1000`, `boolean remap() default true` |
| `@Shadow` [Shadow.java:Shadow] | METHOD, FIELD | RUNTIME | `String prefix() default "shadow$"`, `boolean remap() default true`, `String[] aliases() default {}` |
| `@Overwrite` [Overwrite.java:Overwrite] | METHOD | RUNTIME | `String constraints() default ""`, `String[] aliases() default {}`, `boolean remap() default true` |
| `@Unique` [Unique.java:Unique] | METHOD, FIELD, TYPE | RUNTIME | `boolean silent() default false` |
| `@Final` [Final.java:Final] | METHOD, FIELD | RUNTIME | (marker) |
| `@Mutable` [Mutable.java:Mutable] | METHOD, FIELD | RUNTIME | (marker) |
| `@Debug` [Debug.java:Debug] | TYPE, METHOD | RUNTIME | `boolean export() default false`, `boolean print() default false` |
| `@Dynamic` [Dynamic.java:Dynamic] | METHOD, FIELD | CLASS | `String value() default ""`, `Class<?> mixin() default void.class` |
| `@Implements` [Implements.java:Implements] | TYPE | CLASS | `Interface[] value()` (required) |
| `@Interface` [Interface.java:Interface] | `{}` (nested use only) | CLASS | `Class<?> iface()` (required), `String prefix()` (required), `boolean unique() default false`, `Remap remap() default Remap.ALL` |
| `@Intrinsic` [Intrinsic.java:Intrinsic] | METHOD | CLASS | `boolean displace() default false` |
| `@Pseudo` [Pseudo.java:Pseudo] | TYPE | CLASS | (marker) |
| `@SoftOverride` [SoftOverride.java:SoftOverride] | METHOD | CLASS | (marker) |

- `Interface.Remap` is `{ ALL, FORCE, ONLY_PREFIXED, NONE }`. `forceRemap()` returns true only for `FORCE` [Interface.java:Remap].

### `Mixins` (configuration registry)

A `final` class with a private constructor [Mixins.java:Mixins].

- `addConfiguration(String)`: no fallback environment and no source [Mixins.java:addConfiguration].
- `addConfiguration(String, IMixinConfigSource)` and `addConfigurations(String[], IMixinConfigSource)`: these fall back to `MixinEnvironment.getDefaultEnvironment()` [Mixins.java:addConfigurations].
- `getConfigs()`: returns the live, mutable `Set<Config>` queue [Mixins.java:getConfigs].
- `getUnvisitedCount()` [Mixins.java:getUnvisitedCount].
- `getMixinsForClass(String)`: returns `ClassInfo.fromCache(name).getAppliedMixins()`, or an empty set [Mixins.java:getMixinsForClass].
- `registerErrorHandlerClass(String)` and `getErrorHandlerClasses()`; the getter returns an unmodifiable live view [Mixins.java:getErrorHandlerClasses].

### `MixinEnvironment` (global runtime state)

Static access:

- `init(Phase)`, `getEnvironment(Phase)`, `getDefaultEnvironment()` and `getCurrentEnvironment()`.
- `getCompatibilityLevel()`, `getMinCompatibilityLevel()` and the `@Deprecated` `setCompatibilityLevel(CompatibilityLevel)`.
- The package-private `gotoPhase(Phase)` [MixinEnvironment.java:gotoPhase].

Instance methods:

- `getPhase()`, `getSide()` and `setSide(Side)`.
- `getOption(Option)`, `setOption(Option, boolean)`, `getOptionValue(Option)` and `getOption(Option, E)`.
- `registerTokenProvider` / `registerTokenProviderClass` and `getToken(String)`.
- `getActiveTransformer()` / `setActiveTransformer(IMixinTransformer)` and `audit()`.
- `getRemappers()`, `getObfuscationContext()` / `setObfuscationContext` and `getRefmapObfuscationContext()`.
- `getVersion()`.

Nested types:

- `Phase` [MixinEnvironment.java:Phase] is a class, not an enum. Its public constants are `PREINIT(0)`, `INIT(1)` and `DEFAULT(2)`; `NOT_INITIALISED(-1)` is package-private. `forName(String)` is an exact, case-sensitive lookup that returns null for no match.
- `Side` [MixinEnvironment.java:Side] is `{ UNKNOWN, CLIENT, SERVER }`. It is detected from `IMixinService.getSideName()`: `"CLIENT"` maps to CLIENT, and `"SERVER"` or `"DEDICATEDSERVER"` map to SERVER.
- `Option` [MixinEnvironment.java:Option] keys are `"mixin." + path`. Examples:
  - `DEBUG_ALL` = `mixin.debug`
  - `DEBUG_EXPORT` = `mixin.debug.export`
  - `DEBUG_VERIFY` = `mixin.debug.verify`
  - `DEBUG_STRICT` = `mixin.debug.strict`
  - `DEBUG_INJECTORS` = `mixin.debug.countInjections`
  - `CHECK_ALL` = `mixin.checks`
  - `HOT_SWAP` = `mixin.hotSwap`
  - `DISABLE_REFMAP` = `mixin.env.disableRefMap`
  - `REFMAP_REMAP_SOURCE_ENV` = `mixin.env.refMapRemappingEnv` (default `"searge"`)
  - `DEFAULT_COMPATIBILITY_LEVEL` = `mixin.env.compatLevel`
  - `SHIFT_BY_VIOLATION_BEHAVIOUR` = `mixin.env.shiftByViolation` (default `"warn"`)
  - `INITIALISER_INJECTION_MODE` = `mixin.initialiserInjectionMode` (default `"default"`)
  - `CLASSREADER_EXPAND_FRAMES` = `mixin.tunable.classReaderExpandFrames`

  Each option uses one of four inheritance modes: `INHERIT`, `ALLOW_OVERRIDE`, `INDEPENDENT` or `ALWAYS_FALSE`.
- `CompatibilityLevel` [MixinEnvironment.java:CompatibilityLevel] runs from `JAVA_6` to `JAVA_21`. The static, non-final fields are `DEFAULT = JAVA_6` and `MAX_SUPPORTED = JAVA_13`. Methods: `supports(int)`, `isAtLeast`, `isLessThan`, `canElevateTo`, `canSupport`, `requiredFor(int)` and `getMaxEffective()`.
- `Feature` [MixinEnvironment.java:Feature] is `{ UNSAFE_INJECTION, INJECTORS_IN_INTERFACE_MIXINS }`, with `get`, `exists` and `isActive`.

### Subpackages

- [injection](./injection/EGM.md): the injector annotations (`@Inject`, `@Redirect`, `@ModifyArg` and others), injection points, selectors and callback info.
- [transformer](./transformer/EGM.md): the mixin transformer, mixin configs, the `ClassInfo` cache and the application pipeline.
- [connect](./connect/EGM.md): connector entry points.
- [extensibility](./extensibility/EGM.md): extension SPIs (config plugin, error handler, token provider, remapper, mixin info).
- [gen](./gen/EGM.md): `@Accessor` and `@Invoker` generation.
- [refmap](./refmap/EGM.md): reference maps and refmap remapping.
- [struct](./struct/EGM.md): annotated-method and member-reference data structures.
- [throwables](./throwables/EGM.md): public Mixin exception types.

## Invariants

### Phase lifecycle

- `init(phase)` takes effect only while `currentPhase == NOT_INITIALISED`; later calls are no-ops. It sets `currentPhase` before building the environment and does not roll back if construction throws [MixinEnvironment.java:init].
- `getCurrentEnvironment()` lazily calls `init(PREINIT)` the first time [MixinEnvironment.java:getCurrentPhase].
- There is one `MixinEnvironment` per `Phase`, cached on the phase. Asking for the `NOT_INITIALISED` environment throws `IllegalArgumentException("Cannot access the NOT_INITIALISED environment")` [MixinEnvironment.java:Phase.getEnvironment].
- `gotoPhase(null)`, or a phase with a negative ordinal, throws `IllegalArgumentException("Cannot go to the specified phase, phase is null or invalid")`. `IMixinService.beginPhase()` runs only when moving forward. Moving backward is allowed and skips `beginPhase` [MixinEnvironment.java:gotoPhase].
- The constructor throws `MixinException("Environment conflict, mismatched versions or you didn't call MixinBootstrap.init()")` unless the global property `mixin.initialised` equals `MixinBootstrap.VERSION` [MixinEnvironment.java:MixinEnvironment].

### Options

- Boolean option values are snapshotted from `System` properties into a per-environment array when the environment is constructed. Later property changes do not affect `getOption(Option)` [MixinEnvironment.java:MixinEnvironment].
- `getOptionValue` and `getOption(Option, E)` read `System` properties live on every call [MixinEnvironment.java:getOptionValue].
- `REFMAP_REMAP_ALLOW_PERMISSIVE` and `CLASSREADER_EXPAND_FRAMES` have string defaults (`"true"` / `"false"`). The boolean read still defaults to `false` when the property is unset [MixinEnvironment.java:Option.getBooleanValue].
- `OBFUSCATION_TYPE` and its parent `ENVIRONMENT` are `ALWAYS_FALSE`. So `getRefmapObfuscationContext()` always falls through to the instance's `obfuscationContext` [MixinEnvironment.java:getRefmapObfuscationContext].

### Compatibility level

- `getCompatibilityLevel()` is computed once and cached statically: `max(mixin.env.compatLevel, service min level)`. An invalid property value falls back to the minimum [MixinEnvironment.java:getCompatibilityLevel].
- `setCompatibilityLevel` only raises the level; lower or equal levels are ignored silently. It throws `IllegalArgumentException` when `!level.isSupported()`. A level above the service maximum is applied, with only a warning [MixinEnvironment.java:setCompatibilityLevel].
- `CompatibilityLevel.maxCompatibleLevel` is never assigned, so `canElevateTo` and `canSupport` always return `true`. `isAtLeast(null)` and `isLessThan(null)` are both `true` [MixinEnvironment.java:CompatibilityLevel].
- `getMaxEffective()` is clamped to the range `JAVA_6` through `MAX_SUPPORTED` [MixinEnvironment.java:CompatibilityLevel.getMaxEffective].
- `Feature.INJECTORS_IN_INTERFACE_MIXINS.isEnabled()` depends only on whether the current compatibility level supports `METHODS_IN_INTERFACES` [MixinEnvironment.java:Feature].

### Side, transformer and tokens

- `setSide` changes the side only when the current side is `UNKNOWN` and the argument is neither null nor `UNKNOWN` [MixinEnvironment.java:setSide].
- The active transformer is static, shared by every environment, and a `null` value is ignored [MixinEnvironment.java:setActiveTransformer].
- Token providers are de-duplicated by class name. They sort by priority (higher first), and equal priorities sort the later registration first. `getToken` upper-cases its key with `Locale.ROOT` [MixinEnvironment.java:TokenProviderWrapper.compareTo].

### Config registration

- `Mixins` de-duplicates configs by name through a static set that is never cleared. Re-adding a name is a silent no-op for the lifetime of the classloader [Mixins.java:registerConfiguration].
- A child config is queued before its parent, and parents are registered recursively [Mixins.java:registerConfiguration].
- `Config.create` failures are caught and logged at error level, never rethrown [Mixins.java:createConfiguration].
- A `null` array passed to `addConfigurations` throws `NullPointerException` [Mixins.java:addConfigurations].

### Annotation retention

- `@Mixin`, `@Dynamic`, `@Implements`, `@Interface`, `@Intrinsic`, `@Pseudo` and `@SoftOverride` have CLASS retention, so tests must read them from bytecode, not through reflection [Mixin.java:Mixin].

## Side Effects

- Loading the `Mixins` or `MixinEnvironment` class calls `MixinService.getService().getLogger("mixin")`, so a working `IMixinService` must exist first [Mixins.java:logger], [MixinEnvironment.java:logger].
- `Mixins.getConfigs()` stores a `LinkedHashSet` in global properties under `mixin.configs.queue`. `MixinEnvironment.getMixinConfigs()` stores an `ArrayList` under `mixin.configs.<phase>` [Mixins.java:CONFIGS_KEY], [MixinEnvironment.java:getMixinConfigs].
- The first environment constructed in the JVM logs the `SpongePowered MIXIN Subsystem Version=...` banner. When verbose, export or profiler debugging is on, it also prints an options table to `System.err` [MixinEnvironment.java:printHeader].
- `init` toggles `Profiler.setActive` and wires a `PhaseConsumer` into `MixinServiceAbstract` services. `gotoPhase(DEFAULT)` unwires it [MixinEnvironment.java:init], [MixinEnvironment.java:gotoPhase].
- `registerTokenProviderClass` loads and instantiates a class through the service class provider. It logs failures as errors and does not rethrow [MixinEnvironment.java:registerTokenProviderClass].
- Deprecated methods (`addConfiguration`, `getTransformers`, `addTransformerExclusion`, and `setCompatibilityLevel` when not called from `MixinConfig`) log deprecation warnings [MixinEnvironment.java:setCompatibilityLevel].
- None of this static state (current phase, compatibility, header latch, registered config names, error handlers) has a reset method. Tests need a fresh classloader per scenario [MixinEnvironment.java:currentPhase], [Mixins.java:registeredConfigs].

## Verification

- Run: `mise exec -- ./gradlew compileJava`
- Run: `mise exec -- ./gradlew test`. The `test` source set is declared in `build.gradle`, but `src/test` does not exist yet, so there are no tests today.
