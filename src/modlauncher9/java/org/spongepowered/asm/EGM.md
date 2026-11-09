# modlauncher9: org.spongepowered.asm

## Overview

This directory groups the code specific to ModLauncher 9 and later.

## Subdirectories & Boundaries

- launch: [./launch/EGM.md](./launch/EGM.md). Contains `MixinTransformationService`, whose `completeScan` adds a `mixin_synthetic` GAME-layer jar, and `MixinLaunchPlugin`, which provides the `NamedPath[]` `initializeLaunch`. Its `platform/container` child holds `ContainerHandleModLauncherEx`.
