# launchwrapper: org.spongepowered.asm.service.mojang

## Overview

This package implements the Mixin service for Mojang LegacyLauncher (`net.minecraft.launchwrapper`). It provides the core service contracts in src/main/java/org/spongepowered/asm/service/: `IMixinService`, `IMixinServiceBootstrap` and `IGlobalPropertyService`. Three `META-INF/services` files in this source set register the implementations: `src/launchwrapper/resources/META-INF/services/org.spongepowered.asm.service.IMixinService` (`MixinServiceLaunchWrapper`), `...IMixinServiceBootstrap` (`MixinServiceLaunchWrapperBootstrap`) and `...IGlobalPropertyService` (`Blackboard`). Core `MixinService` in src/main/java/org/spongepowered/asm/service/MixinService.java does the following:

1. Runs every bootstrap, and logs at debug any that throw `ServiceInitialisationException`.
2. Returns the first `IMixinService` whose `isValid()` is true.
3. Uses the first `IGlobalPropertyService` that loads.

## Interface

- `MixinServiceLaunchWrapper extends MixinServiceAbstract implements IClassProvider, IClassBytecodeProvider, ITransformerProvider` [MixinServiceLaunchWrapper.java:class]
  - Constants `BLACKBOARD_KEY_TWEAKCLASSES = Keys.of("TweakClasses")` and `BLACKBOARD_KEY_TWEAKS = Keys.of("Tweaks")` [MixinServiceLaunchWrapper.java:BLACKBOARD_KEY_TWEAKCLASSES], [MixinServiceLaunchWrapper.java:BLACKBOARD_KEY_TWEAKS].
  - `getName()` returns `"LaunchWrapper"`. `isValid()` is true unless `Launch.classLoader.hashCode()` throws [MixinServiceLaunchWrapper.java:getName], [MixinServiceLaunchWrapper.java:isValid].
  - `prepare()`, `getInitialPhase()`, `init()`, `beginPhase()`, `checkEnv(Object)` [MixinServiceLaunchWrapper.java:prepare], [MixinServiceLaunchWrapper.java:getInitialPhase], [MixinServiceLaunchWrapper.java:init], [MixinServiceLaunchWrapper.java:beginPhase], [MixinServiceLaunchWrapper.java:checkEnv].
  - `getMaxCompatibilityLevel()` returns `CompatibilityLevel.JAVA_8`. The minimum level is inherited from `MixinServiceAbstract`, which returns `null` [MixinServiceLaunchWrapper.java:getMaxCompatibilityLevel].
  - `getPlatformAgents()` returns `["org.spongepowered.asm.launch.platform.MixinPlatformAgentFMLLegacy", "org.spongepowered.asm.launch.platform.MixinPlatformAgentLiteLoaderLegacy"]` [MixinServiceLaunchWrapper.java:getPlatformAgents].
  - `getPrimaryContainer()` returns a `ContainerHandleURI` for this class's code source. On `URISyntaxException` or a null URI it falls back to `ContainerHandleVirtual("LaunchWrapper")` [MixinServiceLaunchWrapper.java:getPrimaryContainer].
  - `getMixinContainers()` returns the class-path containers followed by the agent containers [MixinServiceLaunchWrapper.java:getMixinContainers], [MixinServiceLaunchWrapper.java:getContainersFromClassPath].
  - `getClassProvider()`, `getBytecodeProvider()` and `getTransformerProvider()` return `this`. `getClassTracker()` returns the `LaunchClassLoaderUtil`, and `getAuditTrail()` returns `null` [MixinServiceLaunchWrapper.java:getClassProvider], [MixinServiceLaunchWrapper.java:getClassTracker], [MixinServiceLaunchWrapper.java:getAuditTrail].
  - Class lookup:
    - `findClass(String)` calls `Launch.classLoader.findClass`.
    - `findClass(String, boolean)` calls `Class.forName(name, init, Launch.classLoader)`.
    - `findAgentClass(String, boolean)` calls `Class.forName(name, init, Launch.class.getClassLoader())`.

    [MixinServiceLaunchWrapper.java:findClass(String)], [MixinServiceLaunchWrapper.java:findAgentClass]
  - `getResourceAsStream(String)` calls `Launch.classLoader.getResourceAsStream`. `getClassPath()` is deprecated and returns `Launch.classLoader.getSources()` [MixinServiceLaunchWrapper.java:getResourceAsStream], [MixinServiceLaunchWrapper.java:getClassPath].
  - Transformer access:
    - `getTransformers()` wraps each LaunchWrapper transformer that is not already an `ITransformer` in a `LegacyTransformerHandle`.
    - `getDelegatedTransformers()` returns an unmodifiable, cached delegation list.
    - `addTransformerExclusion(String)` adds a name to the exclusion set.

    [MixinServiceLaunchWrapper.java:getTransformers], [MixinServiceLaunchWrapper.java:getDelegatedTransformers], [MixinServiceLaunchWrapper.java:addTransformerExclusion]
  - Bytecode access: `getClassBytes(String, String)` and `getClassBytes(String, boolean)` are deprecated. The `getClassNode` overloads are `(String)`, `(String, boolean)` and `(String, boolean, int)` [MixinServiceLaunchWrapper.java:getClassBytes(String,boolean)], [MixinServiceLaunchWrapper.java:getClassNode(String)].
- `MixinServiceLaunchWrapperBootstrap implements IMixinServiceBootstrap`. `getName()` returns `"LaunchWrapper"`, `getServiceClassName()` returns `"org.spongepowered.asm.service.mojang.MixinServiceLaunchWrapper"`, and `bootstrap()` performs the checks below [MixinServiceLaunchWrapperBootstrap.java:bootstrap].
- `Blackboard implements IGlobalPropertyService`. `resolveKey(String)` returns a `Key` whose `toString()` is the raw name. It also provides `getProperty(key)`, `getProperty(key, default)`, `setProperty(key, value)` and `getPropertyString(key, default)`, all backed by `Launch.blackboard` [Blackboard.java:resolveKey], [Blackboard.java:getProperty(IPropertyKey,T)].
- `final class LaunchClassLoaderUtil implements IClassTracker` is package-private. It provides `isClassLoaded(String)`, `getClassRestrictions(String)` and `registerInvalidClass(String)` [LaunchClassLoaderUtil.java:class].
- `class LegacyTransformerHandle implements ILegacyClassTransformer` is package-private and wraps an `IClassTransformer` [LegacyTransformerHandle.java:class].
- `LoggerAdapterLog4j2 extends LoggerAdapterAbstract`. It is backed by `LogManager.getLogger(name)`, and `getType()` returns `"Log4j2 (via LaunchWrapper)"` [LoggerAdapterLog4j2.java:getType].

## Invariants

- **Missing LaunchWrapper.**
  - `bootstrap()` throws `ServiceInitialisationException("LaunchWrapper is not available")` when `Launch.classLoader.hashCode()` throws, and adds no exclusions [MixinServiceLaunchWrapperBootstrap.java:bootstrap].
  - The `MixinServiceLaunchWrapper` constructor dereferences `Launch.classLoader` without a guard. When LaunchWrapper is absent, the constructor fails during `ServiceLoader` iteration before `isValid()` runs. Core catches that failure, either as a `ServiceConfigurationError` or as another `Throwable`, and moves to the next provider [MixinServiceLaunchWrapper.java:MixinServiceLaunchWrapper()].
  - The `Blackboard()` constructor also calls `Launch.classLoader.hashCode()` without a guard [Blackboard.java:Blackboard()].
- **Initial phase.** `getInitialPhase()` returns `Phase.DEFAULT` only if the stack frame for `net.minecraft.launchwrapper.Launch.launch` is past line 132. Otherwise it returns `Phase.PREINIT`. `findInStackTrace` returns `0` when the thread is not named `main` [MixinServiceLaunchWrapper.java:getInitialPhase], [MixinServiceLaunchWrapper.java:findInStackTrace].
- **Init too early.** `init()` logs an error, without throwing, `"MixinBootstrap.doInit() called during a tweak constructor!"` when that `Launch.launch` frame line is below 4 [MixinServiceLaunchWrapper.java:init].
- **Wrong class loader.** `checkEnv(bootSource)` throws `MixinException("Attempted to init the mixin environment in the wrong classloader")` unless the boot source was loaded by `Launch.class.getClassLoader()` [MixinServiceLaunchWrapper.java:checkEnv].
- **Delegation list.** It contains the `ILegacyClassTransformer` entries of `getTransformers()`, minus any transformer that matches either rule:
  - Its name contains (by substring) an entry of the static `excludeTransformers` set. That set starts with the FML `EventSubscriptionTransformer` and `TerminalTransformer`, in both the `net.minecraftforge.fml.common.asm.transformers.` and the `cpw.mods.fml.common.asm.transformers.` packages.
  - Its `isDelegationExcluded()` is true.

  The list is cached until `beginPhase()` or `addTransformerExclusion()` resets it [MixinServiceLaunchWrapper.java:buildTransformerDelegationList], [MixinServiceLaunchWrapper.java:excludeTransformers].
- **Annotation exclusion.** `LegacyTransformerHandle.isDelegationExcluded()` is true if the wrapped transformer class carries `javax.annotation.Resource`. If that annotation class cannot be found, it returns `false` [LegacyTransformerHandle.java:isDelegationExcluded].
- **Class bytes lookup.** `getClassBytes(String, boolean)` builds the names and fetches bytes as follows:
  1. It converts `/` to `.` to get `transformedName`.
  2. It unmaps that through the last `IClassNameTransformer` registered on `Launch.classLoader` to get `name`.
  3. It reads bytes from `Launch.classLoader.getClassBytes(name)`, falling back to `Launch.class.getClassLoader()` resources.
  4. It runs the delegated transformers when asked.
  5. It throws `ClassNotFoundException("The specified class '<name>' was not found")` if the result is null.

  `getClassNode(String)` always runs the transformers and uses `ClassReader.EXPAND_FRAMES` [MixinServiceLaunchWrapper.java:getClassBytes(String,boolean)], [MixinServiceLaunchWrapper.java:getClassBytes(String,String)].
- **Excluded classes.** `applyTransformers` returns bytes untransformed for classes whose name starts with an entry in LaunchClassLoader's `classLoaderExceptions` or `transformerExceptions` [MixinServiceLaunchWrapper.java:applyTransformers], [LaunchClassLoaderUtil.java:isClassExcluded].
- **Re-entrant transformers.** A transformer that re-enters is excluded permanently, with an INFO log [MixinServiceLaunchWrapper.java:applyTransformers].
- **Class-path containers.** `getContainersFromClassPath` keeps only `file:` URLs that exist and whose manifest `TweakClass` equals `org.spongepowered.asm.launch.MixinTweaker` [MixinServiceLaunchWrapper.java:getContainersFromClassPath].
- **Blackboard semantics.** Keys map 1:1 to `Launch.blackboard` string keys. A stored `null` reads as absent, so the default is returned [Blackboard.java:getProperty(IPropertyKey,T)].
- **Reflected fields.** `LaunchClassLoaderUtil` reads LaunchClassLoader's private fields `cachedClasses`, `invalidClasses`, `classLoaderExceptions` and `transformerExceptions`. A reflection failure leaves that field `null`, and `isClassLoaded` does not null-check `cachedClasses` [LaunchClassLoaderUtil.java:getField], [LaunchClassLoaderUtil.java:isClassLoaded].
- **Class restrictions.** `getClassRestrictions` returns `""`, `PACKAGE_CLASSLOADER_EXCLUSION`, `PACKAGE_TRANSFORMER_EXCLUSION`, or both values joined by a comma [LaunchClassLoaderUtil.java:getClassRestrictions].

## Side Effects

- `bootstrap()` calls `Launch.classLoader.addClassLoaderExclusion` for these packages, in order: `org.spongepowered.asm.service.`, `org.spongepowered.asm.launch.`, `org.spongepowered.asm.logging.`, `org.objectweb.asm.`, `org.spongepowered.asm.lib.`, `org.spongepowered.asm.mixin.`, `org.spongepowered.asm.util.` [MixinServiceLaunchWrapperBootstrap.java:bootstrap].
- `prepare()` adds the class loader exclusion `org.spongepowered.asm.launch.` [MixinServiceLaunchWrapper.java:prepare].
- `getInitialPhase()` sets the system property `mixin.env.remapRefMap=true` when `sun.java.command` contains `GradleStart` [MixinServiceLaunchWrapper.java:getInitialPhase].
- `init()` appends `org.spongepowered.asm.mixin.EnvironmentStateTweaker` to the blackboard `TweakClasses` list, if that list exists. It then calls `super.init()`, which runs each platform agent's `init()` [MixinServiceLaunchWrapper.java:init].
- Every `beginPhase()` calls `Launch.classLoader.registerTransformer("org.spongepowered.asm.mixin.transformer.Proxy")` and clears the delegation cache [MixinServiceLaunchWrapper.java:beginPhase].
- `addTransformerExclusion` permanently mutates the static exclusion set [MixinServiceLaunchWrapper.java:addTransformerExclusion].
- `getClassBytes(String, boolean)` records `Profiler` sections `class.load` and `class.transform`. When `Launch.class.getClassLoader()` is not a `URLClassLoader`, it may create a `URLClassLoader` to read the resource [MixinServiceLaunchWrapper.java:getClassBytes(String,boolean)], [MixinServiceLaunchWrapper.java:getClassBytes(String,String)].
- `Blackboard.setProperty` writes to the global `Launch.blackboard` map [Blackboard.java:setProperty].
- `LaunchClassLoaderUtil` calls `setAccessible(true)` on LaunchClassLoader's private fields, and `registerInvalidClass` writes into its `invalidClasses` set [LaunchClassLoaderUtil.java:getField], [LaunchClassLoaderUtil.java:registerInvalidClass].

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileLaunchwrapperJava`. This uses a Java 8 toolchain with source and target `1.8` (build.gradle lines 124-128) against `net.minecraft:launchwrapper:1.11` (build.gradle lines 181-185).
