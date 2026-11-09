# org.spongepowered.asm.mixin.extensibility

## Overview

This package holds the public extension interfaces that mixin consumers implement (config plugins, error handlers, token providers and remappers). It also holds the read-only views Mixin hands to those consumers (configs, mixin info, config sources and the activity context). Every file is an interface or an interface with a nested enum. The package has no implementation code, static state or I/O. The implementations live in src/main/java/org/spongepowered/asm/mixin/transformer and src/main/java/org/spongepowered/asm/mixin.

## Interface

- `IMixinConfigPlugin`: a companion plugin for a config. Per its javadoc, the config JSON names it under the `plugin` key [IMixinConfigPlugin.java:IMixinConfigPlugin]. Methods:
  - `void onLoad(String mixinPackage)`: called after the plugin is created [IMixinConfigPlugin.java:onLoad]
  - `String getRefMapperConfig()`: called only when the config does not set `referenceMap`. Returning null falls back to the default refmap [IMixinConfigPlugin.java:getRefMapperConfig]
  - `boolean shouldApplyMixin(String targetClassName, String mixinClassName)`: returning false removes that target from the mixin's target set [IMixinConfigPlugin.java:shouldApplyMixin]
  - `void acceptTargets(Set<String> myTargets, Set<String> otherTargets)`: `myTargets` is a live view. Removing a key suppresses this config's mixins for that class, and adding keys has no effect. `otherTargets` is read-only [IMixinConfigPlugin.java:acceptTargets]
  - `List<String> getMixins()`: extra mixin class names to load, or null for none [IMixinConfigPlugin.java:getMixins]
  - `void preApply(String targetClassName, ClassNode targetClass, String mixinClassName, IMixinInfo mixinInfo)` and `postApply(...)`, with the same parameters: called just before and just after a mixin is applied [IMixinConfigPlugin.java:preApply] [IMixinConfigPlugin.java:postApply]
- `IMixinErrorHandler`:
  - `ErrorAction onPrepareError(IMixinConfig config, Throwable th, IMixinInfo mixin, ErrorAction action)` [IMixinErrorHandler.java:onPrepareError]
  - `ErrorAction onApplyError(String targetClassName, Throwable th, IMixinInfo mixin, ErrorAction action)` [IMixinErrorHandler.java:onApplyError]
  - Nested `enum ErrorAction { NONE(Level.INFO), WARN(Level.WARN), ERROR(Level.FATAL) }` with a `public final Level logLevel` field [IMixinErrorHandler.java:ErrorAction]
- `IMixinConfig`: `DEFAULT_PRIORITY = 1000`, `getEnvironment()`, `getName()`, `getSource()`, `getCleanSourceId()`, `getMixinPackage()`, `getPriority()`, `getPlugin()`, `isRequired()`, `getTargets()`, `<V> void decorate(String key, V value)`, `hasDecoration(String)` and `<V> V getDecoration(String)` [IMixinConfig.java:IMixinConfig]
- `IMixinInfo`: `getConfig()`, `getName()`, `getClassName()`, `getClassRef()`, `byte[] getClassBytes()`, `isDetachedSuper()`, `ClassNode getClassNode(int flags)`, `List<String> getTargetClasses()`, `getPriority()` and `Phase getPhase()` [IMixinInfo.java:IMixinInfo]
- `IMixinConfigSource`: `String getId()` and `String getDescription()` [IMixinConfigSource.java:IMixinConfigSource]. `IContainerHandle` in launch/platform/container extends this interface.
- `IEnvironmentTokenProvider`: `DEFAULT_PRIORITY = 1000`, `int getPriority()` and `Integer getToken(String token, MixinEnvironment env)` [IEnvironmentTokenProvider.java:IEnvironmentTokenProvider]
- `IRemapper`: `mapMethodName(owner, name, desc)`, `mapFieldName(owner, name, desc)`, `map(typeName)`, `unmap(typeName)`, `mapDesc(desc)` and `unmapDesc(desc)`, all returning `String` [IRemapper.java:IRemapper]. `RemapperChain` in src/main/java/org/spongepowered/asm/obfuscation implements it.
- `IActivityContext`: `String toString(String glue)`, `IActivity begin(String descriptionFormat, Object... args)`, `IActivity begin(String description)` and `void clear()`. Nested `IActivity` has `next(...)`, `end()` and `append(...)` [IActivityContext.java:IActivityContext] [IActivityContext.java:IActivity]

## Invariants

- `ErrorAction` maps to logging levels as follows: `NONE` to `Level.INFO`, `WARN` to `Level.WARN` and `ERROR` to `Level.FATAL` [IMixinErrorHandler.java:ErrorAction].
- Error handler contract, per the javadoc: returning null or the passed-in `action` keeps the default behaviour, and returning a different `ErrorAction` changes the severity. The javadoc says the default is `ERROR` for configs marked required and `WARN` otherwise [IMixinErrorHandler.java:onPrepareError] [IMixinErrorHandler.java:onApplyError].
- `IEnvironmentTokenProvider.getToken` receives the token in upper case and returns null when the provider has no value [IEnvironmentTokenProvider.java:getToken].
- `IMixinConfig.decorate` is documented to throw `IllegalArgumentException` when the key already exists. `getDecoration` returns null when the key is absent [IMixinConfig.java:decorate] [IMixinConfig.java:getDecoration].
- `IMixinConfig.getCleanSourceId` is documented to return null when the source or its id is null [IMixinConfig.java:getCleanSourceId].
- `IMixinConfig.getSource` may return null when the service supplied no source [IMixinConfig.java:getSource].

## Side Effects

- None. This package contains only interface declarations and a constant-backed enum.

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileJava`
- Run from the repo root: `mise exec -- ./gradlew test`
- `build.gradle` declares a `test` source set, but `src/test` does not exist yet. `test` therefore runs no tests today.
