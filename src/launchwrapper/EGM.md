# launchwrapper source set

## Overview

The `launchwrapper` Gradle source set integrates Mixin with Mojang LegacyLauncher (`net.minecraft:launchwrapper:1.11`, build.gradle lines 181-185). It is compiled with a Java 8 toolchain at source and target `1.8` against the `main` output (build.gradle lines 124-128). Its classes go into the published jar, which merges every non-example, non-test source set and their service files (`stagingJar` with `mergeServiceFiles()`, build.gradle lines 350-374). The source set has no `modularityExcluded` flag, so its output is also patched into the `org.spongepowered.mixin` module (build.gradle lines 312-341).

### How LegacyLauncher discovers Mixin

- **Entry point.** The tweaker `org.spongepowered.asm.launch.MixinTweaker` is named on the command line or in a manifest `TweakClass` attribute.
- **Service discovery.** Core `MixinService` uses `java.util.ServiceLoader` over these files in `resources/META-INF/services/`:
  - `org.spongepowered.asm.service.IMixinServiceBootstrap` lists `org.spongepowered.asm.service.mojang.MixinServiceLaunchWrapperBootstrap`.
  - `org.spongepowered.asm.service.IMixinService` lists `org.spongepowered.asm.service.mojang.MixinServiceLaunchWrapper`.
  - `org.spongepowered.asm.service.IGlobalPropertyService` lists `org.spongepowered.asm.service.mojang.Blackboard`.
- **Validity.** The bootstrap and the service are usable only when `net.minecraft.launchwrapper.Launch.classLoader` is reachable. When it is not:
  - The bootstrap throws `ServiceInitialisationException("LaunchWrapper is not available")`, which core logs at debug and skips.
  - The `MixinServiceLaunchWrapper` constructor fails while `ServiceLoader` is iterating. Core catches that failure, either as a `ServiceConfigurationError` or as another `Throwable`, and tries the next registered `IMixinService` (src/main/java/org/spongepowered/asm/service/MixinService.java).

## Subdirectories & Boundaries

- java: [./java/EGM.md](./java/EGM.md). The Java sources: `MixinTweaker`, `EnvironmentStateTweaker`, `Proxy`, the FML and LiteLoader platform agents, and the `"LaunchWrapper"` service with its `Launch.blackboard` property service.

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileLaunchwrapperJava`.
