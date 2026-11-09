# agent source set

## Overview

The `agent` Gradle source set holds Mixin's hot-swap agent,
`org.spongepowered.tools.agent.MixinAgent`, and its package-private
`MixinAgentClassLoader`.

How the agent is started:

- The `stagingJar` manifest names `MixinAgent` as both `Premain-Class` and
  `Agent-Class`, with `Can-Redefine-Classes: true` and
  `Can-Retransform-Classes: true` (build.gradle:363-367). The Mixin jar can
  therefore be passed to `-javaagent` or attached to a running JVM.
- `MixinTransformer.initHotSwapper`
  (src/main/java/org/spongepowered/asm/mixin/transformer/MixinTransformer.java)
  creates the agent by class name when `mixin.hotSwap` is on.

Build and packaging:

- The source set is declared at build.gradle:105-109. It compiles against
  `main.output` with `languageVersion = 8` and `compatibility = '1.6'`.
- `agentImplementation` extends `implementation` (build.gradle:77).
- src/modularity/java/module-info.java exports `org.spongepowered.tools.agent`.

## Subdirectories & Boundaries

- java: [java](./java/EGM.md) - Java source root. The agent reloads redefined
  mixins through `IMixinTransformer.reload`. It then redefines each target class
  with `Instrumentation.redefineClasses`, starting from the target's stored
  pre-mixin bytecode.

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileAgentJava`.
