# java (fernflower source set)

## Overview

Java source root of the `fernflower` Gradle source set (build.gradle:99-104).

## Subdirectories & Boundaries

- org: [org](./org/EGM.md) - Runtime decompiler for the debug class export. `RuntimeDecompiler` (synchronous) and `RuntimeDecompilerAsync` (one daemon thread) use Fernflower to write `<outputPath>/<qualifiedName>.java` as UTF-8, and the constructor deletes the existing output directory. `ExtensionClassExporter` (src/main) loads them by class name when `mixin.debug.export.decompile` is on.
