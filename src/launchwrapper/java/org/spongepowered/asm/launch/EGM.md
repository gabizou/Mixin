# launchwrapper: org.spongepowered.asm.launch

## Overview

This package holds `MixinTweaker`, the LegacyLauncher `ITweaker` that boots Mixin. Clients name it with `--tweakClass org.spongepowered.asm.launch.MixinTweaker` or with a jar manifest `TweakClass` attribute; `MixinServiceLaunchWrapper` scans for that manifest value. Each `ITweaker` callback maps to one phase of core `MixinBootstrap` (src/main/java/org/spongepowered/asm/launch/MixinBootstrap.java). The `platform` child package holds the FML and LiteLoader agents.

## Interface

- `public class MixinTweaker implements net.minecraft.launchwrapper.ITweaker` [MixinTweaker.java:MixinTweaker]
  - `public MixinTweaker()` calls `MixinBootstrap.start()` [MixinTweaker.java:MixinTweaker()].
  - `public final void acceptOptions(List<String> args, File gameDir, File assetsDir, String profile)` calls `MixinBootstrap.doInit(CommandLineOptions.ofArgs(args))`. The other three arguments are unused [MixinTweaker.java:acceptOptions].
  - `public final void injectIntoClassLoader(LaunchClassLoader)` calls `MixinBootstrap.inject()`. The class loader argument is unused [MixinTweaker.java:injectIntoClassLoader].
  - `String getLaunchTarget()` returns `MixinBootstrap.getPlatform().getLaunchTarget()` [MixinTweaker.java:getLaunchTarget].
  - `String[] getLaunchArguments()` always returns an empty array [MixinTweaker.java:getLaunchArguments].
- Child package: [platform](./platform/EGM.md) holds `MixinPlatformAgentFMLLegacy` and `MixinPlatformAgentLiteLoaderLegacy`.

## Invariants

- **Boot order under LaunchWrapper.** LaunchWrapper fixes the callback order; the core files named here are cited by path.
  1. **Tweaker construction.** `new MixinTweaker()` calls `MixinBootstrap.start()` [MixinTweaker.java:MixinTweaker()].
     - Initializing the `MixinBootstrap` class runs `MixinService.boot()`, which executes `MixinServiceLaunchWrapperBootstrap.bootstrap()`.
     - It then calls `MixinService.getService().prepare()`.
     - `start()` then reads `getInitialPhase()`, calls `MixinEnvironment.init(...)`, then `beginPhase()`. That first `beginPhase()` registers the first `Proxy` transformer.
  2. **`acceptOptions`.** It calls `MixinBootstrap.doInit(...)`, which calls `getPlatform().prepare(args)` and then `MixinService.getService().init()`. `MixinServiceLaunchWrapper.init()` queues `org.spongepowered.asm.mixin.EnvironmentStateTweaker` in blackboard `TweakClasses` [MixinTweaker.java:acceptOptions].
  3. **`injectIntoClassLoader`.** It calls `MixinBootstrap.inject()`, which calls `getPlatform().inject()` [MixinTweaker.java:injectIntoClassLoader].
  4. **`EnvironmentStateTweaker`.** It runs later as a queued tweaker. It re-injects the platform, then moves Mixin to `Phase.DEFAULT` (see the `mixin` package).
- `MixinBootstrap.doInit` throws `IllegalStateException("MixinBootstrap.doInit() called before MixinBootstrap.start()")`, but only when `start()` did not run and no Mixin subsystem is already registered (src/main/java/org/spongepowered/asm/launch/MixinBootstrap.java).
- `acceptOptions` and `injectIntoClassLoader` are `final`. `MixinTweaker` catches no exceptions, so bootstrap failures reach LaunchWrapper [MixinTweaker.java:acceptOptions].

## Side Effects

- Constructing the tweaker starts the Mixin subsystem, which runs the service bootstraps and adds class loader exclusions to `Launch.classLoader` [MixinTweaker.java:MixinTweaker()].
- `acceptOptions` parses LaunchWrapper's argument list into `CommandLineOptions` [MixinTweaker.java:acceptOptions].

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileLaunchwrapperJava`. This uses a Java 8 toolchain with source and target `1.8` (build.gradle lines 124-128) against `net.minecraft:launchwrapper:1.11` (build.gradle lines 181-185).
