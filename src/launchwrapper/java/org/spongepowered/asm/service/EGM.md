# launchwrapper: org.spongepowered.asm.service

## Overview

This directory groups the Mojang LegacyLauncher implementation of the Mixin service contracts. The contracts themselves (`IMixinService`, `IMixinServiceBootstrap`, `IGlobalPropertyService`, `MixinServiceAbstract`) live in the main source set at src/main/java/org/spongepowered/asm/service/.

## Subdirectories & Boundaries

- mojang: [./mojang/EGM.md](./mojang/EGM.md). Contains `MixinServiceLaunchWrapper` (service name `"LaunchWrapper"`), `MixinServiceLaunchWrapperBootstrap`, the `Launch.blackboard`-backed `Blackboard`, `LaunchClassLoaderUtil`, `LegacyTransformerHandle` and `LoggerAdapterLog4j2`. The service is valid only when `Launch.classLoader` is reachable. Each `beginPhase()` registers the `Proxy` transformer.
