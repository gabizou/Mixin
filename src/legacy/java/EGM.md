# java (legacy source set)

## Overview

Java source root of the `legacy` Gradle source set (build.gradle:85-88).

## Subdirectories & Boundaries

- org: [org](./org/EGM.md) - Binary-compatibility shims for 0.7.x companion plugins: the shim `org.spongepowered.asm.lib.ClassVisitor`, five empty placeholder types, `org.spongepowered.asm.lib.tree.ClassNode`, and `CompanionPluginError`. Every shim method throws `CompanionPluginError`.
