# bridge source set

## Overview

The `bridge` Gradle source set holds `org.spongepowered.asm.bridge`. These
adapters wrap an ASM `org.objectweb.asm.commons.Remapper`, notably Forge's
`FMLDeobfuscatingRemapper`, as Mixin's `IRemapper`. The only consumer is
`MixinPlatformAgentFMLLegacy`
(src/launchwrapper/java/org/spongepowered/asm/launch/platform/MixinPlatformAgentFMLLegacy.java).
It loads `RemapperAdapterFML` by class name and adds the result of `create()`
to the default environment's remappers.

Build and packaging:

- The source set is declared at build.gradle:110-115. It compiles against
  `main.output` with `languageVersion = 8`, `compatibility = '1.8'` and
  `modularityExcluded = true`.
- It does not extend `implementation`. Its dependencies are
  `org.apache.logging.log4j:log4j-core:2.0-beta9` and
  `org.ow2.asm:asm-commons:$legacyForgeAsmVersion` (build.gradle:211-212).
- That ASM version resolves to 9.5, not 5.0.3. build.gradle:42 returns
  `asmVersion` whenever the `legacyForgeAsmVersion` property is set, and
  gradle.properties:10 sets it.
- `stagingJar` packages the classes into the main jar (build.gradle:350-353).

## Subdirectories & Boundaries

- java: [java](./java/EGM.md) - Java source root. `RemapperAdapter` maps method
  and field names, retrying with the owner and descriptor unmapped.
  `RemapperAdapterFML` gets its `unmap` from the FML remapper by reflection.

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileBridgeJava`.
