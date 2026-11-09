# launchwrapper: org.spongepowered.asm.mixin

## Overview

This package holds `EnvironmentStateTweaker`, a LegacyLauncher `ITweaker` that `MixinServiceLaunchWrapper.init()` queues in blackboard `TweakClasses`. It runs after the earlier tweakers and moves Mixin into `Phase.DEFAULT`. The `transformer` child package holds the `Proxy` class transformer.

## Interface

- `public class EnvironmentStateTweaker implements net.minecraft.launchwrapper.ITweaker`, with an implicit no-argument constructor [EnvironmentStateTweaker.java:EnvironmentStateTweaker]
  - `acceptOptions(List<String>, File, File, String)` does nothing [EnvironmentStateTweaker.java:acceptOptions].
  - `injectIntoClassLoader(LaunchClassLoader)` calls `MixinBootstrap.getPlatform().inject()` [EnvironmentStateTweaker.java:injectIntoClassLoader].
  - `getLaunchTarget()` returns `""` [EnvironmentStateTweaker.java:getLaunchTarget].
  - `getLaunchArguments()` calls `MixinEnvironment.gotoPhase(Phase.DEFAULT)`, then returns `new String[0]` [EnvironmentStateTweaker.java:getLaunchArguments].
- Child package: [transformer](./transformer/EGM.md) holds the `Proxy` `IClassTransformer`.

## Invariants

- The move to `Phase.DEFAULT` happens when LaunchWrapper calls `getLaunchArguments()`, not when it calls `injectIntoClassLoader()` [EnvironmentStateTweaker.java:getLaunchArguments].
- Core `MixinEnvironment.gotoPhase` calls `service.beginPhase()` only when the target phase is later than the current one (src/main/java/org/spongepowered/asm/mixin/MixinEnvironment.java). Under LaunchWrapper, that `beginPhase()` registers a new `Proxy`, which deactivates every earlier proxy.
- The tweaker holds no state and catches no exceptions [EnvironmentStateTweaker.java:EnvironmentStateTweaker].

## Side Effects

- `injectIntoClassLoader` runs `inject()` on the platform manager a second time [EnvironmentStateTweaker.java:injectIntoClassLoader].
- `getLaunchArguments` changes the global `MixinEnvironment` phase [EnvironmentStateTweaker.java:getLaunchArguments].

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileLaunchwrapperJava`. This uses a Java 8 toolchain with source and target `1.8` (build.gradle lines 124-128) against `net.minecraft:launchwrapper:1.11` (build.gradle lines 181-185).
