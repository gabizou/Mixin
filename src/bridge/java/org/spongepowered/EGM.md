# org.spongepowered (bridge source set)

## Overview

Package path segment in the `bridge` Gradle source set.

## Subdirectories & Boundaries

- asm: [asm](./asm/EGM.md) - Adapters from an ASM `Remapper` to Mixin's `IRemapper` and `IClassRemapper`. `RemapperAdapterFML.create()` wraps Forge's `FMLDeobfuscatingRemapper.INSTANCE` by reflection and returns `null` on failure. The LaunchWrapper FML platform agent (src/launchwrapper) loads it by class name.
