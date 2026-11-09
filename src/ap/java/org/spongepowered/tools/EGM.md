# ap: org/spongepowered/tools

## Overview

Package grouping in the `ap` source set. It has no source files and contains
only the `obfuscation` package, which holds the Mixin annotation processor.

## Subdirectories & Boundaries

- obfuscation: [./obfuscation/EGM.md](./obfuscation/EGM.md). The Mixin
  annotation processor:
  - Two processors: `MixinObfuscationProcessorInjection` and
    `MixinObfuscationProcessorTargets`.
  - The option keys in `SupportedOptions`.
  - The refmap written to `outRefMapFile`.
  - SRG and TSRG mapping files written for each obfuscation type.
  - Obfuscation services discovered with `ServiceLoader`, plus the
    `MessageType` diagnostics and the parent and target validators.
