# org.spongepowered.asm (fernflower source set)

## Overview

Package path segment in the `fernflower` Gradle source set.

## Subdirectories & Boundaries

- mixin: [mixin](./mixin/EGM.md) - Runtime decompiler for the debug class export. `RuntimeDecompiler` (synchronous) and `RuntimeDecompilerAsync` (one daemon thread) use Fernflower to write `<outputPath>/<qualifiedName>.java` as UTF-8, and the constructor deletes the existing output directory. `ExtensionClassExporter` (src/main) loads them by class name when `mixin.debug.export.decompile` is on.
