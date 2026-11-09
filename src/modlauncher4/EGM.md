# modlauncher4 source set

## Overview

The `modlauncher4` Gradle source set supplies the ModLauncher 4-8 transformation service, and all class-path `ServiceLoader` registrations for the ModLauncher integration. It is compiled with a Java 8 toolchain at source and target `1.8`, against the `main` and `modlauncher` outputs (build.gradle lines 134-139) and `cpw.mods:modlauncher:7.0.0` (build.gradle lines 197-203). The `stagingJar` task merges these service files into the published jar (`mergeServiceFiles()`, build.gradle lines 350-374).

### How ModLauncher 4-8 discovers Mixin

Each file in `resources/META-INF/services/` names one class:

| Service file | Registered class |
| --- | --- |
| `cpw.mods.modlauncher.api.ITransformationService` | `org.spongepowered.asm.launch.MixinTransformationServiceLegacy` |
| `cpw.mods.modlauncher.serviceapi.ILaunchPluginService` | `org.spongepowered.asm.launch.MixinLaunchPluginLegacy` |
| `org.spongepowered.asm.service.IMixinServiceBootstrap` | `org.spongepowered.asm.service.modlauncher.MixinServiceModLauncherBootstrap` |
| `org.spongepowered.asm.service.IMixinService` | `org.spongepowered.asm.service.modlauncher.MixinServiceModLauncher` |
| `org.spongepowered.asm.service.IGlobalPropertyService` | `org.spongepowered.asm.service.modlauncher.Blackboard` |

The launch plugin and transformation service classes listed here are the legacy ones. The ModLauncher 9+ classes, `MixinTransformationService` and `MixinLaunchPlugin`, are registered only through `provides` clauses in `src/modularity/java/module-info.java`. The `modlauncher9` source set has no `resources/` folder.

## Subdirectories & Boundaries

- java: [./java/EGM.md](./java/EGM.md). Contains `MixinTransformationServiceLegacy`, which extends the shared `MixinTransformationServiceAbstract` with a no-op `beginScanning`.

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileModlauncher4Java`.
