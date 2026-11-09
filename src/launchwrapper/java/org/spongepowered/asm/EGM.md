# launchwrapper: org.spongepowered.asm

## Overview

This directory groups the Mixin packages that adapt Mixin to Mojang LegacyLauncher: the tweaker entry point, the transformer proxy, the environment-state tweaker and the service implementation.

## Subdirectories & Boundaries

- launch: [./launch/EGM.md](./launch/EGM.md). `MixinTweaker` maps the `ITweaker` callbacks to `MixinBootstrap.start()`, `doInit()` and `inject()`. Its `platform` child holds the FML and LiteLoader agents.
- mixin: [./mixin/EGM.md](./mixin/EGM.md). `EnvironmentStateTweaker` moves Mixin to `Phase.DEFAULT` in `getLaunchArguments()`. Its `transformer` child holds `Proxy`, where only the newest instance transforms.
- service: [./service/EGM.md](./service/EGM.md). The `IMixinService`, `IMixinServiceBootstrap` and `IGlobalPropertyService` implementations for LaunchWrapper.
