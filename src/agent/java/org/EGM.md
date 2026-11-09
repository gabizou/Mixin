# org (agent source set)

## Overview

Package path segment in the `agent` Gradle source set.

## Subdirectories & Boundaries

- spongepowered: [spongepowered](./spongepowered/EGM.md) - Hot-swap `java.lang.instrument` agent. `MixinAgent` (the manifest `Premain-Class` and `Agent-Class`) reloads a redefined mixin and redefines its target classes from their stored pre-mixin bytecode. `MixinAgentClassLoader` holds the stub mixin classes and that original target bytecode.
