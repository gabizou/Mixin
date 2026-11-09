# modlauncher9: org.spongepowered.asm.launch.platform

## Overview

This directory has no source files of its own. It groups the ModLauncher 9+ container package.

## Subdirectories & Boundaries

- container: [./container/EGM.md](./container/EGM.md). Contains `ContainerHandleModLauncherEx`, the root container used when the ModLauncher API spec version is at least `8.0`. It adds each `SecureJar` resource as a `SecureJarResource` child and delegates other resource types to `ContainerHandleModLauncher`.
