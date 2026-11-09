# fernflower source set

## Overview

The `fernflower` Gradle source set holds the runtime decompiler used by Mixin's
debug class export: `RuntimeDecompiler` and `RuntimeDecompilerAsync`, both in
`org.spongepowered.asm.mixin.transformer.debug`. The main source set refers to it only
through the `IDecompiler` interface, whose Javadoc says it exists "to allow the
actual decompiler to be loaded on-demand".
`ExtensionClassExporter`
(src/main/java/org/spongepowered/asm/mixin/transformer/ext/extensions/ExtensionClassExporter.java)
loads the decompiler by class name. If loading fails, it logs at INFO and
disables decompilation.

Build and packaging:

- The source set is declared at build.gradle:99-104. It compiles against
  `main.output` with `languageVersion = 8`, `compatibility = '1.6'` and
  `modularityExcluded = true`.
- `fernflowerImplementation` extends `implementation` (build.gradle:75) and adds
  `org.jetbrains.java.decompiler:fernflower:sponge-SNAPSHOT` (build.gradle:178).
  The artifact comes from https://repo.spongepowered.org/maven
  (build.gradle:59-63).
- `stagingJar` packages the classes into the main jar (build.gradle:350-353).

## Subdirectories & Boundaries

- java: [java](./java/EGM.md) - Java source root. When it is enabled, the
  decompiler deletes and rewrites `.mixin.out/java`.

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileFernflowerJava`.
