# modlauncher9 source set

## Overview

The `modlauncher9` Gradle source set supplies the cpw ModLauncher 9+ transformation service, launch plugin and root container. It is compiled with a Java 16 toolchain, with no source or target override, against the `main` and `modlauncher` outputs (build.gradle lines 140-144). Its dependencies are `cpw.mods:modlauncher:10.0.9` (the `modlauncherVersion` in gradle.properties) and `cpw.mods:securejarhandler:2.1.24` (build.gradle lines 205-208).

### How ModLauncher 9+ discovers Mixin

This source set has no `resources/` folder and no `META-INF/services` files. When Mixin is loaded as the named module `org.spongepowered.mixin`, discovery goes through the `provides` clauses in `src/modularity/java/module-info.java`:

| Service | Provider |
| --- | --- |
| `cpw.mods.modlauncher.api.ITransformationService` | `org.spongepowered.asm.launch.MixinTransformationService` |
| `cpw.mods.modlauncher.serviceapi.ILaunchPluginService` | `org.spongepowered.asm.launch.MixinLaunchPlugin` |
| `IMixinServiceBootstrap`, `IMixinService` and `IGlobalPropertyService` | `MixinServiceModLauncherBootstrap`, `MixinServiceModLauncher` and `Blackboard`, all in `src/modlauncher` |

At runtime, `MixinServiceModLauncher` selects `ContainerHandleModLauncherEx` and the minimum level `JAVA_16` when the ModLauncher API spec version is at least `8.0`.

## Subdirectories & Boundaries

- java: [./java/EGM.md](./java/EGM.md). Contains `MixinTransformationService` (adds the `mixin_synthetic` GAME-layer jar), `MixinLaunchPlugin` (`NamedPath[]` `initializeLaunch`) and `ContainerHandleModLauncherEx` (`SecureJar` resources).

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileModlauncher9Java`.
