# org.spongepowered.asm (bridge source set)

## Overview

This directory has no source files of its own. In this source set it holds only the `bridge` package.

## Subdirectories & Boundaries

- bridge: [bridge](./bridge/EGM.md) - Adapters from an ASM `Remapper` to Mixin's `IRemapper` and `IClassRemapper`. `RemapperAdapterFML.create()` wraps Forge's `FMLDeobfuscatingRemapper.INSTANCE` by reflection and returns `null` on failure. The LaunchWrapper FML platform agent (src/launchwrapper) loads it by class name.
