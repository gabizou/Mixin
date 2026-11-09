# java (bridge source set)

## Overview

Java source root of the `bridge` Gradle source set (build.gradle:110-115).

## Subdirectories & Boundaries

- org: [org](./org/EGM.md) - Adapters from an ASM `Remapper` to Mixin's `IRemapper` and `IClassRemapper`. `RemapperAdapterFML.create()` wraps Forge's `FMLDeobfuscatingRemapper.INSTANCE` by reflection and returns `null` on failure. The LaunchWrapper FML platform agent (src/launchwrapper) loads it by class name.
