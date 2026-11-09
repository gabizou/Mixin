# launchwrapper: org.spongepowered.asm.launch.platform

## Overview

These are the platform agents for Mojang LegacyLauncher (launchwrapper). `MixinServiceLaunchWrapper.getPlatformAgents()` names both of them. `MixinPlatformAgentFMLLegacy` lets Mixin containers act as legacy Forge Mod Loader (FML) coremods. `MixinPlatformAgentLiteLoaderLegacy` only reports the side from LiteLoader. Both extend core `MixinPlatformAgentAbstract` and implement `IMixinPlatformServiceAgent` (src/main/java/org/spongepowered/asm/launch/platform/). Both reach FML and LiteLoader classes only by reflection or through `GlobalProperties`, so they load and run when those mods are absent.

## Interface

- `MixinPlatformAgentFMLLegacy` [MixinPlatformAgentFMLLegacy.java:MixinPlatformAgentFMLLegacy]
  - `AcceptResult accept(MixinPlatformManager, IContainerHandle)`: decides whether a container is an FML coremod [MixinPlatformAgentFMLLegacy.java:accept].
  - `void prepare()`, `void inject()`, `void init()` [MixinPlatformAgentFMLLegacy.java:prepare], [MixinPlatformAgentFMLLegacy.java:inject], [MixinPlatformAgentFMLLegacy.java:init].
  - `String getSideName()` [MixinPlatformAgentFMLLegacy.java:getSideName].
  - `Collection<IContainerHandle> getMixinContainers()` always returns `null` [MixinPlatformAgentFMLLegacy.java:getMixinContainers].
  - `String getPhaseProvider()` returns `"org.spongepowered.asm.launch.platform.MixinPlatformAgentFMLLegacy$PhaseProvider"`. This file declares no nested `PhaseProvider` class [MixinPlatformAgentFMLLegacy.java:getPhaseProvider].
  - `@Deprecated void wire(Phase, IConsumer<Phase>)` and `@Deprecated void unwire()` [MixinPlatformAgentFMLLegacy.java:wire], [MixinPlatformAgentFMLLegacy.java:unwire].
  - `protected final boolean checkForCoInitialisation()` [MixinPlatformAgentFMLLegacy.java:checkForCoInitialisation].
  - Package-private `static class MixinAppender extends AbstractAppender` (Log4j2 core), named `"MixinLogWatcherAppender"` [MixinPlatformAgentFMLLegacy.java:MixinAppender].
- `MixinPlatformAgentLiteLoaderLegacy` [MixinPlatformAgentLiteLoaderLegacy.java:MixinPlatformAgentLiteLoaderLegacy]
  - `accept(...)` always returns `AcceptResult.REJECTED` [MixinPlatformAgentLiteLoaderLegacy.java:accept].
  - `getSideName()` returns `invokeStringMethod(Launch.classLoader, "com.mumfrey.liteloader.launch.LiteLoaderTweaker", "getEnvironmentType")` [MixinPlatformAgentLiteLoaderLegacy.java:getSideName].
  - `init()` does nothing, and `getMixinContainers()` returns `null` [MixinPlatformAgentLiteLoaderLegacy.java:init], [MixinPlatformAgentLiteLoaderLegacy.java:getMixinContainers].

### String literals and keys used by the FML agent

| Literal | Purpose | Source |
| --- | --- | --- |
| `net.minecraftforge.fml.relauncher.CoreModManager`, then `cpw.mods.fml.relauncher.CoreModManager` | CoreModManager lookup (the first is overridable by the global property `mixin.launch.fml.coremodmanagerclass`) | [MixinPlatformAgentFMLLegacy.java:getCoreModManagerClass] |
| `loadCoreMod` (global property `mixin.launch.fml.loadcoremodmethod`) | Coremod loader method | [MixinPlatformAgentFMLLegacy.java:injectCorePlugin] |
| `getReparseableCoremods` (global property `mixin.launch.fml.reparseablecoremodsmethod`) | List of reparseable jars | [MixinPlatformAgentFMLLegacy.java:addReparseableJar] |
| `getIgnoredMods` (global property `mixin.launch.fml.ignoredmodsmethod`), then `getLoadedCoremods` | List of ignored mods | [MixinPlatformAgentFMLLegacy.java:getIgnoredMods] |
| Manifest `FMLCorePlugin`, `ForceLoadAsMod`, `FMLCorePluginContainsFMLMod` | Coremod container attributes | [MixinPlatformAgentFMLLegacy.java:MFATT_FMLCOREPLUGIN] |
| System property `fml.coreMods.load` | Comma-separated coremods to ignore | [MixinPlatformAgentFMLLegacy.java:FML_CMDLINE_COREMODS] |
| Blackboard `Tweaks` and `TweakClasses` | LaunchWrapper tweaker list and queued tweak classes | [MixinPlatformAgentFMLLegacy.java:isAlreadyInjected], [MixinPlatformAgentFMLLegacy.java:isTweakerQueued] |
| `FMLDeobfTweaker`, `FMLInjectionAndSortingTweaker`, `TerminalTweaker` | Tweaker names matched by suffix | [MixinPlatformAgentFMLLegacy.java:checkForCoInitialisation] |
| `org.spongepowered.asm.bridge.RemapperAdapterFML` with factory method `create` | FML remapper bridge | [MixinPlatformAgentFMLLegacy.java:injectRemapper] |
| Log4j logger `FML`, message `Validating minecraft` | FML INIT-phase trigger | [MixinPlatformAgentFMLLegacy.java:begin], [MixinPlatformAgentFMLLegacy.java:MixinAppender] |
| `net.minecraftforge.fml.relauncher.FMLLaunchHandler`, then `cpw.mods.fml.relauncher.FMLLaunchHandler`, method `side` | Side fallback | [MixinPlatformAgentFMLLegacy.java:getSideName] |

The global property key strings are defined in src/main/java/org/spongepowered/asm/launch/GlobalProperties.java. The blackboard keys are defined in [MixinServiceLaunchWrapper.java:BLACKBOARD_KEY_TWEAKS] and [MixinServiceLaunchWrapper.java:BLACKBOARD_KEY_TWEAKCLASSES].

## Invariants

- `accept` checks, in order:
  1. It returns `INVALID` if `getCoreModManagerClass()` is `null`.
  2. It returns `REJECTED` if the handle is not a `ContainerHandleURI`, or if `super.accept` is not `ACCEPTED`.
  3. It runs `initFMLCoreMod()` and returns `ACCEPTED` only if that produced a coremod wrapper. A container without an `FMLCorePlugin` attribute is therefore `REJECTED` [MixinPlatformAgentFMLLegacy.java:accept], [MixinPlatformAgentFMLLegacy.java:injectCorePlugin].
- If `ForceLoadAsMod` is `"true"` (case-insensitive), `loadAsMod()` runs before `injectCorePlugin()`. `loadAsMod()` removes the jar from FML's ignored-mods list. It also re-adds the jar as reparseable when `FMLCorePluginContainsFMLMod` has any non-null value, unless the handle's `toString()` contains `deobfedDeps` [MixinPlatformAgentFMLLegacy.java:initFMLCoreMod], [MixinPlatformAgentFMLLegacy.java:loadAsMod].
- A coremod is not loaded twice. `isAlreadyInjected` is true if the name is in the static `loadedCoreMods` set, which is seeded from `-Dfml.coreMods.load`. It is also true if an `FMLPluginWrapper` tweaker in blackboard `Tweaks` has a `coreModInstance` of that class name. It returns `false` when `Tweaks` is null, and on any exception [MixinPlatformAgentFMLLegacy.java:isAlreadyInjected].
- `checkForCoInitialisation()` returns `false` when `FMLInjectionAndSortingTweaker` is queued, or when `TerminalTweaker` is queued while the injection state is set. In those cases FML injects the coremod itself. Otherwise it returns `true` only if `FMLDeobfTweaker` is not queued. `inject()` calls the coremod wrapper's `injectIntoClassLoader(Launch.classLoader)` only when this returns `true` [MixinPlatformAgentFMLLegacy.java:checkForCoInitialisation], [MixinPlatformAgentFMLLegacy.java:inject].
- `isTweakerQueued` matches by `endsWith` against blackboard `TweakClasses`. It does not null-check that list, so a missing list throws `NullPointerException` [MixinPlatformAgentFMLLegacy.java:isTweakerQueued].
- When FML is missing, `getCoreModManagerClass()` logs at INFO `"FML platform manager could not load class {}. Proceeding without FML support."` and returns `null`. The null is not cached, so the lookup is retried on every call. `accept` then returns `INVALID`, and `init()` skips the remapper [MixinPlatformAgentFMLLegacy.java:getCoreModManagerClass], [MixinPlatformAgentFMLLegacy.java:init].
- A missing or failing `RemapperAdapterFML` is logged at debug and swallowed (`catch (Exception)`) [MixinPlatformAgentFMLLegacy.java:injectRemapper].
- `getSideName()` returns `null` if blackboard `Tweaks` is null. Otherwise it returns `SERVER` for a tweaker ending in `.common.launcher.FMLServerTweaker` and `CLIENT` for one ending in `.common.launcher.FMLTweaker`, checking server first. If no tweaker matches, it calls `FMLLaunchHandler.side()` in the new package, then the old one [MixinPlatformAgentFMLLegacy.java:getSideName].
- `wire` installs the log appender only when `phase == Phase.PREINIT`. The appender calls `delegate.accept(Phase.INIT)` on every DEBUG `Validating minecraft` event, with no once-only guard. If the `FML` logger is not a Log4j2 core `Logger`, no appender is installed [MixinPlatformAgentFMLLegacy.java:wire], [MixinPlatformAgentFMLLegacy.java:begin], [MixinPlatformAgentFMLLegacy.java:MixinAppender].
- `MixinPlatformAgentLiteLoaderLegacy` never accepts a container. Its only LiteLoader dependency is the reflective side lookup [MixinPlatformAgentLiteLoaderLegacy.java:accept].

## Side Effects

- The static initializer reads `System.getProperty("fml.coreMods.load", "")` and adds each entry to the static `loadedCoreMods` set, which is never cleared [MixinPlatformAgentFMLLegacy.java:loadedCoreMods].
- It reflectively calls FML `CoreModManager.loadCoreMod(Launch.classLoader, name, file)`, mutates FML's ignored-mods and reparseable-coremods lists, and adds the name to `loadedCoreMods` [MixinPlatformAgentFMLLegacy.java:injectCorePlugin], [MixinPlatformAgentFMLLegacy.java:loadAsMod], [MixinPlatformAgentFMLLegacy.java:addReparseableJar].
- It adds the `IRemapper` from `RemapperAdapterFML.create()` to `MixinEnvironment.getDefaultEnvironment().getRemappers()` [MixinPlatformAgentFMLLegacy.java:injectRemapper].
- It attaches `MixinAppender` to the `FML` Log4j2 logger and sets that logger's level to `ALL`. `unwire()` and `end()` remove the appender but do not restore the level [MixinPlatformAgentFMLLegacy.java:begin], [MixinPlatformAgentFMLLegacy.java:end].

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileLaunchwrapperJava`. This uses a Java 8 toolchain with source and target `1.8` (build.gradle lines 124-128) against `net.minecraft:launchwrapper:1.11` (build.gradle lines 181-185).
