# ap source set (Mixin annotation processor)

## Overview

The Gradle `ap` source set holds the compile-time annotation processor for
Mixin. While mixin classes compile, it generates refmaps and obfuscation
mapping files.

From `build.gradle`:

- The compile classpath includes `main.output`.
- Dependencies are `org.ow2.asm:asm-tree` and Guava.
- The source set has `languageVersion = 8` and `compatibility = '1.8'`.
- Its output goes into `stagingJar`, the main jar, which merges service files.
- It is the only content of the `shadowJar` with the classifier `processor`.

The source set also contains `src/ap/resources`, which has no code. It holds
two `ServiceLoader` registration files:

- `META-INF/services/javax.annotation.processing.Processor` lists
  `org.spongepowered.tools.obfuscation.MixinObfuscationProcessorInjection` and
  `org.spongepowered.tools.obfuscation.MixinObfuscationProcessorTargets`.
  `javac` reads it to find the processors.
- `META-INF/services/org.spongepowered.tools.obfuscation.service.IObfuscationService`
  lists `org.spongepowered.tools.obfuscation.mcp.ObfuscationServiceMCP` and
  `org.spongepowered.tools.obfuscation.fg3.ObfuscationServiceFG3`.
  `ObfuscationServices` reads it.

## Subdirectories & Boundaries

- java: [./java/EGM.md](./java/EGM.md). Holds the processor code in
  `org.spongepowered.tools.obfuscation`:
  - Option keys such as `outRefMapFile`, `tokens`, `defaultObfuscationEnv`,
    `mappingTypes`, `quiet` and `MSG_<TYPE>`.
  - The refmap JSON, written to `outRefMapFile` as a path or a `CLASS_OUTPUT`
    resource.
  - Mapping files for each type: `outSrgFile`, `outNotchSrgFile` and
    `outTsrgFile`.
  - `MessageType` diagnostics, which `@SuppressWarnings` tokens can suppress.
  - The EARLY, LATE and FINAL validation passes.
  - Static state that lasts as long as the class loader and carries over
    between compilations in one JVM.

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileApJava`
- Run: `mise exec -- ./gradlew test`. Note that `src/test` does not exist yet,
  so no tests cover the processor today.
