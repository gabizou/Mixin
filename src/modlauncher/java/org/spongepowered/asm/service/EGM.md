# modlauncher: org.spongepowered.asm.service

## Overview

This directory groups the cpw ModLauncher implementation of the Mixin service contracts. The contracts themselves (`IMixinService`, `IMixinServiceBootstrap`, `IGlobalPropertyService`, `MixinServiceAbstract`) live in the main source set at src/main/java/org/spongepowered/asm/service/.

## Subdirectories & Boundaries

- modlauncher: [./modlauncher/EGM.md](./modlauncher/EGM.md). Contains `MixinServiceModLauncher` (service name `"ModLauncher"`, valid from API spec version `4.0`) and its bootstrap. It also holds the two `IClassProcessor`s, `MixinTransformationHandler` and `ModLauncherClassTracker`, plus the class provider, the audit trail, the `TypesafeMap` `Blackboard` and the Log4j2 adapter.
