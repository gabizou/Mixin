# modularity: java

## Overview

This directory holds `module-info.java`, the JPMS descriptor for the module `org.spongepowered.mixin`. Gradle compiles it with `--patch-module`, using the outputs of every source set not marked `modularityExcluded`. That includes `main`, `launchwrapper`, `modlauncher`, `modlauncher4` and `modlauncher9`. This descriptor is how ModLauncher 9+ discovers Mixin when Mixin is on the module path.

## Interface

- `module org.spongepowered.mixin` [module-info.java:module]
- `requires transitive`: `java.compiler`, `java.instrument`, `org.objectweb.asm`, `org.objectweb.asm.commons`, `org.objectweb.asm.tree`, `org.objectweb.asm.tree.analysis`, `org.objectweb.asm.util`. Plain `requires java.logging` [module-info.java:requires].
- `requires static` (compile-time only):
  - `org.apache.logging.log4j.core`, `cpw.mods.securejarhandler` and `jopt.simple`.
  - The transitive pair `cpw.mods.modlauncher` and `org.apache.logging.log4j`.
  - Guava and Gson under both module names: `com.google.common` and `guava`, `com.google.gson` and `gson`.

  [module-info.java:requires static]
- 46 unqualified `exports`, including `org.spongepowered.asm.launch`, `org.spongepowered.asm.launch.platform`, `org.spongepowered.asm.launch.platform.container`, `org.spongepowered.asm.service` and `org.spongepowered.asm.service.modlauncher`. `org.spongepowered.asm.service.mojang` is not exported [module-info.java:exports].
- `opens org.spongepowered.asm.mixin.transformer to com.google.gson, gson`. That package is opened but not exported [module-info.java:opens].
- Service declarations, each with a matching `uses` [module-info.java:provides]:

| Service | `provides ... with` |
| --- | --- |
| `org.spongepowered.asm.service.IMixinServiceBootstrap` | `org.spongepowered.asm.service.modlauncher.MixinServiceModLauncherBootstrap` |
| `org.spongepowered.asm.service.IMixinService` | `org.spongepowered.asm.service.modlauncher.MixinServiceModLauncher` |
| `org.spongepowered.asm.service.IGlobalPropertyService` | `org.spongepowered.asm.service.modlauncher.Blackboard` |
| `cpw.mods.modlauncher.api.ITransformationService` | `org.spongepowered.asm.launch.MixinTransformationService` |
| `cpw.mods.modlauncher.serviceapi.ILaunchPluginService` | `org.spongepowered.asm.launch.MixinLaunchPlugin` |
| `javax.annotation.processing.Processor` | `org.spongepowered.tools.obfuscation.MixinObfuscationProcessorInjection`, `org.spongepowered.tools.obfuscation.MixinObfuscationProcessorTargets` |
| `org.spongepowered.tools.obfuscation.service.IObfuscationService` | `org.spongepowered.tools.obfuscation.mcp.ObfuscationServiceMCP`, `org.spongepowered.tools.obfuscation.fg3.ObfuscationServiceFG3` |

## Invariants

- The module descriptor registers only the ModLauncher Mixin service. It provides no LaunchWrapper service classes, so `MixinServiceLaunchWrapper` is reachable only through `src/launchwrapper/resources/META-INF/services/` [module-info.java:provides].
- On the module path, the ModLauncher 9+ classes `MixinTransformationService` and `MixinLaunchPlugin` are registered. The class-path files in `src/modlauncher4/resources/META-INF/services/` instead register `MixinTransformationServiceLegacy` and `MixinLaunchPluginLegacy` [module-info.java:provides].
- ModLauncher, SecureJarHandler, Log4j, jopt-simple, Guava and Gson are `requires static`, so the module resolves without them at runtime [module-info.java:requires static].

## Side Effects

- None

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileModularityJava`. This uses a Java 16 toolchain (build.gradle lines 145-148), with `--module-path` and `--patch-module org.spongepowered.mixin=<non-excluded source set outputs>` (build.gradle lines 312-341). `modularityImplementation` extends `modlauncher9Implementation` (build.gradle line 80), and `log4j-core:2.11.2` is compile-only (build.gradle line 216).
