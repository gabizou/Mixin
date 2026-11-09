# org.spongepowered.asm.mixin.transformer (fernflower source set)

## Overview

This directory has no source files of its own. It holds the `debug` package. The `transformer` package itself lives in src/main/java/org/spongepowered/asm/mixin/transformer, which refers to `debug` only by class-name string.

## Subdirectories & Boundaries

- debug: [debug](./debug/EGM.md) - Runtime decompiler for the debug class export. `RuntimeDecompiler` (synchronous) and `RuntimeDecompilerAsync` (one daemon thread) use Fernflower to write `<outputPath>/<qualifiedName>.java` as UTF-8, and the constructor deletes the existing output directory. `ExtensionClassExporter` (src/main) loads them by class name when `mixin.debug.export.decompile` is on.
