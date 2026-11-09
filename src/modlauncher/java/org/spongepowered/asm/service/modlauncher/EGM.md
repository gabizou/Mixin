# modlauncher: org.spongepowered.asm.service.modlauncher

## Overview

This package implements the Mixin service for cpw ModLauncher. It provides the core contracts in src/main/java/org/spongepowered/asm/service/. One service class serves both ModLauncher 4-8 and ModLauncher 9+, and picks its root container and minimum Java level from the ModLauncher API specification version. The launch plugin (`MixinLaunchPluginLegacy`, in the sibling `launch` package) drives class processing through the two `IClassProcessor`s returned by `getProcessors()`.

The service is registered in two places:

- On the class path, through `src/modlauncher4/resources/META-INF/services/`.
- As the named module `org.spongepowered.mixin`, through `provides` clauses in `src/modularity/java/module-info.java`.

This source set itself has no `resources/` folder.

## Interface

- `public class MixinServiceModLauncher extends MixinServiceAbstract` [MixinServiceModLauncher.java:class]
  - `getName()` returns `"ModLauncher"` [MixinServiceModLauncher.java:getName].
  - `isValid()` is true when the ModLauncher spec version is at least `4.0` and no `Throwable` occurs [MixinServiceModLauncher.java:isValid].
  - `getMinCompatibilityLevel()` returns `JAVA_8`, or `JAVA_16` when the spec version is at least `8.0`. The maximum level, the initial phase (`Phase.PREINIT`), `prepare()`, `init()`, `beginPhase()`, `getMixinContainers()` and `getSideName()` are inherited from `MixinServiceAbstract` [MixinServiceModLauncher.java:getMinCompatibilityLevel].
  - `void onInit(IClassBytecodeProvider)`, called by the launch plugin's `init` [MixinServiceModLauncher.java:onInit].
  - `void onStartup()`, called by the launch plugin's `initializeLaunch` [MixinServiceModLauncher.java:onStartup].
  - `void wire(Phase, IConsumer<Phase>)`, which stores the phase consumer [MixinServiceModLauncher.java:wire].
  - `void offer(IMixinInternal)`, which passes an `IMixinTransformerFactory` to the transformation handler [MixinServiceModLauncher.java:offer].
  - `Collection<IClassProcessor> getProcessors()` returns `[MixinTransformationHandler, ModLauncherClassTracker]` in that order [MixinServiceModLauncher.java:getProcessors].
  - `getClassProvider()` returns a `ModLauncherClassProvider`. `getClassTracker()` returns a `ModLauncherClassTracker`. `getAuditTrail()` returns a `ModLauncherAuditTrail`. Each is created lazily and cached [MixinServiceModLauncher.java:getClassProvider], [MixinServiceModLauncher.java:getClassTracker], [MixinServiceModLauncher.java:getAuditTrail].
  - `getBytecodeProvider()` returns the provider passed to `onInit` [MixinServiceModLauncher.java:getBytecodeProvider].
  - `getTransformerProvider()` returns `null` [MixinServiceModLauncher.java:getTransformerProvider].
  - `getPlatformAgents()` returns `["org.spongepowered.asm.launch.platform.MixinPlatformAgentMinecraftForge"]` [MixinServiceModLauncher.java:getPlatformAgents].
  - `getPrimaryContainer()` returns a `ContainerHandleModLauncher`, the root container [MixinServiceModLauncher.java:getPrimaryContainer].
  - `getResourceAsStream(String)` reads from the thread context class loader [MixinServiceModLauncher.java:getResourceAsStream].
- `MixinServiceModLauncherBootstrap implements IMixinServiceBootstrap`. `getName()` returns `"ModLauncher"`, and `getServiceClassName()` returns `"org.spongepowered.asm.service.modlauncher.MixinServiceModLauncher"` [MixinServiceModLauncherBootstrap.java:bootstrap].
- `MixinTransformationHandler implements IClassProcessor` provides `handlesClass`, `processClass`, `generatesClass` and `generateClass`, plus the package-private `offer(IMixinTransformerFactory)` [MixinTransformationHandler.java:class].
- `ModLauncherClassTracker implements IClassProcessor, IClassTracker` provides `registerInvalidClass`, `isClassLoaded`, and `getClassRestrictions`, which returns `""` [ModLauncherClassTracker.java:class].
- `ModLauncherClassProvider implements IClassProvider` is package-private [ModLauncherClassProvider.java:class].
  - `findClass(String)` uses `Class.forName(name, true, TCCL)`, where TCCL is the thread context class loader.
  - `findClass(String, boolean)` uses the TCCL with the given `initialize` flag.
  - `findAgentClass` uses `Launcher.class.getClassLoader()`.
  - `getClassPath()` is deprecated. It reflectively calls `cpw.mods.gross.Java9ClassLoaderUtil.getSystemClassPathURLs`, and returns `new URL[0]` on failure.
- `ModLauncherAuditTrail implements IMixinAuditTrail` provides `setConsumer(String, Consumer<String[]>)`, `onApply`, `onPostProcess` and `onGenerate`, which emit the activity codes `"APP"`, `"DEC"` and `"GEN"` [ModLauncherAuditTrail.java:class].
- `Blackboard implements IGlobalPropertyService` is backed by `Launcher.INSTANCE.blackboard()`, a `TypesafeMap` [Blackboard.java:class].
  - `resolveKey(String)` caches one `TypesafeMap.Key<Object>` per name [Blackboard.java:resolveKey].
  - It also provides `getProperty`, `setProperty` and `getPropertyString` [Blackboard.java:setProperty].
- `LoggerAdapterLog4j2` is backed by `LogManager.getLogger(name)`, and `getType()` returns `"Log4j2 (via ModLauncher)"` [LoggerAdapterLog4j2.java:getType].

## Invariants

- **API version and container selection.** The service determines the ModLauncher API specification version in this order:
  1. The `IEnvironment.Keys.MLSPEC_VERSION` property.
  2. The `ITransformationService` package specification version.
  3. `VersionNumber.NONE` if neither is available.

  The constructor then chooses the root container class [MixinServiceModLauncher.java:getModLauncherApiVersion], [MixinServiceModLauncher.java:MixinServiceModLauncher()]:
  - At `8.0` or above, it uses `org.spongepowered.asm.launch.platform.container.ContainerHandleModLauncherEx` and sets the minimum level to `JAVA_16`.
  - Below `8.0`, it uses `...ContainerHandleModLauncher` and keeps `JAVA_8`.
- **Root container creation.** The root container is created with `getClassProvider().findClass(name)` followed by `newInstance("ModLauncher")`. A `ReflectiveOperationException` is printed to stderr and leaves `getPrimaryContainer()` returning `null` [MixinServiceModLauncher.java:createRootContainer].
- **Missing ModLauncher.** `bootstrap()` touches `Launcher.INSTANCE` and turns any `Throwable` into `ServiceInitialisationException("ModLauncher is not available")` [MixinServiceModLauncherBootstrap.java:bootstrap]. The service constructor reads the API version without a guard, so without ModLauncher it fails during `ServiceLoader` iteration. Core then tries the next provider. `isValid()` catches `Throwable` and returns `false` [MixinServiceModLauncher.java:isValid].
- **Call-order guards.**
  - A second `onInit` throws `IllegalStateException("Already initialised")` [MixinServiceModLauncher.java:onInit].
  - `getBytecodeProvider()` throws `IllegalStateException("Service initialisation incomplete")` before `onInit` [MixinServiceModLauncher.java:getBytecodeProvider].
  - `onStartup()` dereferences the consumer that `wire(...)` stores. Core `MixinEnvironment.init` calls `wire` once (src/main/java/org/spongepowered/asm/mixin/MixinEnvironment.java), so `onStartup()` before that throws `NullPointerException` [MixinServiceModLauncher.java:onStartup].
- **`MixinTransformationHandler.handlesClass`.** It returns `Phases.AFTER_ONLY` for non-empty classes. For empty classes it returns `AFTER_ONLY` only if the synthetic class registry knows the class, and `null` before the first `processClass` [MixinTransformationHandler.java:handlesClass].
- **`MixinTransformationHandler.processClass`.** It is `synchronized` and routes in this order [MixinTransformationHandler.java:processClass]:
  1. `Phase.BEFORE` returns `false`.
  2. It creates the transformer from the offered factory if needed. It throws `IllegalStateException("processClass called before transformer factory offered to transformation handler")` if no factory was offered.
  3. Reason `"mixin"` returns `false`.
  4. A synthetic class calls `generateClass`.
  5. Reason `ITransformerActivity.COMPUTING_FRAMES_REASON` calls `computeFramesForClass`.
  6. Anything else calls `transformClass`.
- **`MixinTransformationHandler.generatesClass`.** It dereferences the registry without a null check [MixinTransformationHandler.java:generatesClass].
- **`ModLauncherClassTracker.handlesClass`.** It always returns `AFTER_ONLY`. It throws `NoClassDefFoundError("<name> is invalid")` for names passed to `registerInvalidClass` [ModLauncherClassTracker.java:handlesClass].
- **`ModLauncherClassTracker.isClassLoaded`.** It becomes true only after `processClass` with reason `ITransformerActivity.CLASSLOADING_REASON`. `processClass` always returns `false`, and the tracker never removes names [ModLauncherClassTracker.java:processClass].
- **Audit activity.** An activity reaches ModLauncher only if the consumer is set and the class name equals the class passed to `setConsumer` [ModLauncherAuditTrail.java:writeActivity].
- **Blackboard semantics.**
  - `setProperty` uses `computeIfAbsent`, so the first value written to a key wins [Blackboard.java:setProperty].
  - `getProperty` returns the default for a missing key [Blackboard.java:getProperty].
  - A foreign `IPropertyKey` implementation causes a `ClassCastException` [Blackboard.java:setProperty].

## Side Effects

- The service constructor reads `Launcher.INSTANCE.environment()` and initializes the root container class through the thread context class loader [MixinServiceModLauncher.java:MixinServiceModLauncher()].
- `onStartup()` calls the phase consumer with `Phase.DEFAULT`. Core's `PhaseConsumer` calls `MixinEnvironment.gotoPhase` [MixinServiceModLauncher.java:onStartup].
- `MixinTransformationHandler.processClass` mutates the `ClassNode` that ModLauncher passes in [MixinTransformationHandler.java:processClass].
- `ModLauncherClassTracker` records loaded class names. Its `NoClassDefFoundError` aborts the class load in ModLauncher [ModLauncherClassTracker.java:processClass], [ModLauncherClassTracker.java:handlesClass].
- `ModLauncherAuditTrail` calls ModLauncher's audit `Consumer<String[]>` with `{"APP", mixinName}`, `{"DEC"}` or `{"GEN"}` [ModLauncherAuditTrail.java:onApply].
- `Blackboard` registers keys in, and writes to, ModLauncher's global `TypesafeMap` blackboard [Blackboard.java:resolveKey], [Blackboard.java:setProperty].

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileModlauncherJava`. This uses a Java 8 toolchain with source and target `1.8` (build.gradle lines 129-133) against `cpw.mods:modlauncher:7.0.0` and `jopt-simple:5.0.4`, with `log4j-core:2.0-beta9` as compile-only (build.gradle lines 187-195).
