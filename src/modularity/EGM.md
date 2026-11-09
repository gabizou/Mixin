# modularity source set

## Overview

The `modularity` Gradle source set compiles only the JPMS descriptor for the module `org.spongepowered.mixin`, using a Java 16 toolchain. It is itself marked `modularityExcluded`, so it does not patch itself into the module (build.gradle lines 145-148). Every other source set without that flag adds its output to the modularity compile class path and to `--patch-module` (build.gradle lines 312-341). The `sourceJar` task skips this source set (build.gradle line 424).

## Subdirectories & Boundaries

- java: [./java/EGM.md](./java/EGM.md). `module-info.java` declares `provides` clauses for the ModLauncher Mixin service, its bootstrap, the property service, `MixinTransformationService` and `MixinLaunchPlugin`, plus the annotation processor and obfuscation services. It also exports 46 packages and opens `org.spongepowered.asm.mixin.transformer` to Gson.

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileModularityJava`.
