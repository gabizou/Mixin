# modlauncher: org.spongepowered.asm

## Overview

This directory groups the ModLauncher integration code shared by every supported ModLauncher version: the launch plugin, the base transformation service, the Forge side agent, the root container and the Mixin service.

## Subdirectories & Boundaries

- launch: [./launch/EGM.md](./launch/EGM.md). Contains `MixinTransformationServiceAbstract` (`ITransformationService`, named `"mixin"`), `MixinLaunchPluginLegacy` (`ILaunchPluginService`, named `"mixin"`), `IClassProcessor` and `Phases`. Its children hold the Forge agent and `ContainerHandleModLauncher`.
- service: [./service/EGM.md](./service/EGM.md). Contains the `"ModLauncher"` `IMixinService`, its bootstrap, and the property, class, audit and logging services.
