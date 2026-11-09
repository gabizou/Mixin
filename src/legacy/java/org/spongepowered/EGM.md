# org.spongepowered (legacy source set)

## Overview

Package path segment in the `legacy` Gradle source set.

## Subdirectories & Boundaries

- asm: [asm](./asm/EGM.md) - Binary-compatibility shims for 0.7.x companion plugins: the shim `org.spongepowered.asm.lib.ClassVisitor`, five empty placeholder types, `org.spongepowered.asm.lib.tree.ClassNode`, and `CompanionPluginError`. Every shim method throws `CompanionPluginError`.
