# java (agent source set)

## Overview

Java source root of the `agent` Gradle source set (build.gradle:105-109).

## Subdirectories & Boundaries

- org: [org](./org/EGM.md) - Hot-swap `java.lang.instrument` agent. `MixinAgent` (the manifest `Premain-Class` and `Agent-Class`) reloads a redefined mixin and redefines its target classes from their stored pre-mixin bytecode. `MixinAgentClassLoader` holds the stub mixin classes and that original target bytecode.
