# org.spongepowered.asm (legacy source set)

## Overview

This directory has no source files of its own. It groups the two package trees
of the legacy source set. Both exist only so that `IMixinConfigPlugin`
companions compiled against Mixin 0.7.x still link under 0.8.

## Subdirectories & Boundaries

- lib: [lib](./lib/EGM.md) - These are shims for the 0.7.x repackaged ASM
  names `org.spongepowered.asm.lib.*`. The shim `ClassVisitor` throws
  `CompanionPluginError` from every method. The `tree/ClassNode` shim gives
  read-only access to eleven header fields copied from a native ASM
  `ClassNode`.
- mixin: [mixin](./mixin/EGM.md) - This holds `throwables/CompanionPluginError`,
  the `LinkageError` that every shim throws.
