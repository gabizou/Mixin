# main source set

## Overview

The `main` Gradle source set is the core Mixin runtime that every launcher integration builds on: the consumer annotations, the transformer pipeline, the injector system, bootstrap and the service SPI, and the utilities. All other source sets except `test` and `modularity` compile against `main.output` (`build.gradle`, `sourceSets`).

## Subdirectories & Boundaries

- java: [./java/EGM.md](./java/EGM.md). The Java sources for `org.spongepowered.asm.*`.

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileJava`.
- Run from the repository root: `mise exec -- ./gradlew test`. The `test` source set is declared in `build.gradle`, but `src/test` does not exist yet, so there are no tests today.
