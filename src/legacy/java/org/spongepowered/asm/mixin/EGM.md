# org.spongepowered.asm.mixin (legacy source set)

## Overview

This directory has no source files of its own. Its one subdirectory holds the legacy source set's part of `org.spongepowered.asm.mixin.throwables`. That package is split with the main source set (src/main/java/org/spongepowered/asm/mixin/throwables).

## Subdirectories & Boundaries

- throwables: [throwables](./throwables/EGM.md) - `CompanionPluginError extends LinkageError`, which the legacy shims and `PluginHandle.applyLegacy` (src/main) throw for unsupported legacy API calls.
