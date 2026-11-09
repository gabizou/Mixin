# main: java

## Overview

This is the Java source root of the `main` Gradle source set, the core Mixin runtime. It compiles with a Java 8 toolchain at compatibility `1.6`, against `legacy.output`, ASM `asm-tree`, `asm-commons` and `asm-util` at `$asmVersion`, Guava 21.0 and Gson 2.2.4 (`build.gradle`, `sourceSets.main` and `dependencies`).

## Subdirectories & Boundaries

- org: [./org/EGM.md](./org/EGM.md). The `org.spongepowered.asm` runtime packages.
