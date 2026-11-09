# modlauncher source set

## Overview

The `modlauncher` Gradle source set holds the cpw ModLauncher integration that every supported ModLauncher version shares. It is compiled with a Java 8 toolchain at source and target `1.8` (build.gradle lines 129-133) against `cpw.mods:modlauncher:7.0.0` (the `legacyModlauncherVersion` in gradle.properties) and `jopt-simple:5.0.4`, with `log4j-core:2.0-beta9` as compile-only (build.gradle lines 187-195).

This source set has no `resources/` folder, so it registers nothing by itself. Its classes are discovered through two other source sets:

- **ModLauncher 4-8 on the class path.** `src/modlauncher4/resources/META-INF/services/` registers `MixinTransformationServiceLegacy`, `MixinLaunchPluginLegacy`, `MixinServiceModLauncher`, `MixinServiceModLauncherBootstrap` and the ModLauncher `Blackboard`.
- **ModLauncher 9+ as the named module `org.spongepowered.mixin`.** `src/modularity/java/module-info.java` registers `MixinTransformationService`, `MixinLaunchPlugin` and the same three Mixin service classes.

The published jar merges all of these (`stagingJar` with `mergeServiceFiles()`, build.gradle lines 350-374).

When ModLauncher is missing, `MixinServiceModLauncherBootstrap` throws `ServiceInitialisationException("ModLauncher is not available")`, which core `MixinService` logs at debug and skips.

## Subdirectories & Boundaries

- java: [./java/EGM.md](./java/EGM.md). Contains the launch plugin (`"mixin"`), the base transformation service, `IClassProcessor` and `Phases`, the Forge side agent, `ContainerHandleModLauncher`, and the `"ModLauncher"` Mixin service with its processors and `TypesafeMap` blackboard.

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileModlauncherJava`.
