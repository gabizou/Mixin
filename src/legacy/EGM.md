# legacy source set

## Overview

The `legacy` Gradle source set holds binary-compatibility shims for
`IMixinConfigPlugin` companions compiled against Mixin 0.7.x. In that version
ASM was repackaged as `org.spongepowered.asm.lib`. The shims do no ASM work:

- Every shim method throws `CompanionPluginError`.
- The shim `ClassNode` lets a plugin read eleven header fields of the real
  target class.

Build and packaging:

- The source set is declared at build.gradle:85-88 with `languageVersion = 8`
  and `compatibility = '1.6'`.
- Its only dependency is `legacyImplementation
  "org.ow2.asm:asm-tree:$asmVersion"` (build.gradle:214). It does not extend
  `implementation`.
- The main source set compiles against it (`main { compileClasspath +=
  legacy.output }`, build.gradle:90). Javadoc adds it to its classpath
  (build.gradle:221).
- `stagingJar` packages every source set except `example` and `test`, so these
  classes ship in the main jar (build.gradle:350-353).
- The source set is not `modularityExcluded`. src/modularity/java/module-info.java
  exports `org.spongepowered.asm.lib`, `org.spongepowered.asm.lib.tree` and
  `org.spongepowered.asm.mixin.throwables`.

Consumers outside this partition:

- `PluginHandle` (src/main/java/org/spongepowered/asm/mixin/transformer/PluginHandle.java)
  falls back to this `ClassNode` after a plugin throws `AbstractMethodError`.
  It wraps failures from the legacy call in `CompanionPluginError`.
- `MixinServiceLaunchWrapperBootstrap`
  (src/launchwrapper/java/org/spongepowered/asm/service/mojang/MixinServiceLaunchWrapperBootstrap.java)
  registers `"org.spongepowered.asm.lib."` as a LaunchWrapper class-loader
  exclusion.

## Subdirectories & Boundaries

- java: [java](./java/EGM.md) - Java source root. It holds the packages
  `org.spongepowered.asm.lib`, `org.spongepowered.asm.lib.tree` and
  `org.spongepowered.asm.mixin.throwables`.

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileLegacyJava`.
