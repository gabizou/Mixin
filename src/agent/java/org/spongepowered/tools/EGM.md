# org.spongepowered.tools (agent source set)

## Overview

This directory has no source files of its own. In this source set it holds only the `agent` package.

## Subdirectories & Boundaries

- agent: [agent](./agent/EGM.md) - Hot-swap `java.lang.instrument` agent. `MixinAgent` (the manifest `Premain-Class` and `Agent-Class`) reloads a redefined mixin and redefines its target classes from their stored pre-mixin bytecode. `MixinAgentClassLoader` holds the stub mixin classes and that original target bytecode.
