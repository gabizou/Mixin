# src

## Overview

Each child of `src/` is one Gradle source set declared in `build.gradle` (`sourceSets { ... }`). `main` is the core runtime, and the others compile against `main.output`. `fernflower`, `bridge`, `example`, `test` and `modularity` are marked `ext.modularityExcluded = true`, which leaves them out of the module-info patch. `stagingJar` packages every source set except `example` and `test`. A `test` source set is declared in `build.gradle`, but `src/test/` does not exist.

## Subdirectories & Boundaries

- main: [./main/EGM.md](./main/EGM.md). The core Mixin runtime (`org.spongepowered.asm.*`): annotations, transformer, injectors, bootstrap, service SPI and utilities. Java 8 toolchain, compatibility 1.6.
- legacy: [./legacy/EGM.md](./legacy/EGM.md). Binary-compatibility shims for 0.7.x companion plugins (`org.spongepowered.asm.lib.*`, `CompanionPluginError`). Every shim method throws.
- launchwrapper: [./launchwrapper/EGM.md](./launchwrapper/EGM.md). The Mojang LegacyLauncher integration: the `MixinTweaker` entry point, the `LaunchWrapper` service, the blackboard property service and the `Proxy` class transformer.
- modlauncher: [./modlauncher/EGM.md](./modlauncher/EGM.md). The shared cpw ModLauncher integration: the `ModLauncher` service, `MixinLaunchPluginLegacy`, `MixinTransformationServiceAbstract` and the class processors.
- modlauncher4: [./modlauncher4/EGM.md](./modlauncher4/EGM.md). The ModLauncher 4–8 `MixinTransformationServiceLegacy`, plus every class-path `META-INF/services` registration for ModLauncher.
- modlauncher9: [./modlauncher9/EGM.md](./modlauncher9/EGM.md). The ModLauncher 9+ `MixinTransformationService` (the `mixin_synthetic` GAME-layer jar), `MixinLaunchPlugin` and `ContainerHandleModLauncherEx`. Java 16 toolchain.
- modularity: [./modularity/EGM.md](./modularity/EGM.md). `module-info.java` for the `org.spongepowered.mixin` module. Its `provides` clauses register the ModLauncher 9+ services on the module path.
- ap: [./ap/EGM.md](./ap/EGM.md). The compile-time annotation processor that writes refmaps and obfuscation mapping files, with the MCP (SRG) and ForgeGradle 3 (TSRG) obfuscation services.
- example: [./example/EGM.md](./example/EGM.md). A sample third-party `IObfuscationService` (`json`) for the annotation processor.
- agent: [./agent/EGM.md](./agent/EGM.md). `MixinAgent`, the `java.lang.instrument` hot-swap agent (the manifest's `Premain-Class` and `Agent-Class`).
- fernflower: [./fernflower/EGM.md](./fernflower/EGM.md). `RuntimeDecompiler` and `RuntimeDecompilerAsync`, which back the debug export decompiler and write to `.mixin.out/java`.
- bridge: [./bridge/EGM.md](./bridge/EGM.md). `RemapperAdapter` and `RemapperAdapterFML`, which bridge an ASM `Remapper` to `IRemapper` for legacy FML.
