# org.spongepowered.asm

## Overview

This package is the root of the Mixin runtime in the `main` source set. It declares no classes; `package-info.java` holds only the package Javadoc, "The Mixin runtime and utilities." The subpackages split the runtime into the consumer API and pipeline (`mixin`), bootstrap and platform discovery (`launch`), the host-service SPI (`service`), and support code (`logging`, `obfuscation`, `transformers`, `util`).

## Interface

- `package-info.java`: Javadoc only, no symbols [package-info.java:org.spongepowered.asm].
- [launch](./launch/EGM.md): `MixinBootstrap` (`VERSION = "0.8.7"`, `init`, and the package-private `start`, `doInit` and `inject`), the `GlobalProperties` blackboard facade, `MixinInitialisationError`, and the `platform` layer (`MixinPlatformManager`, `MixinContainer`, platform agents, container handles, `CommandLineOptions`).
- [logging](./logging/EGM.md): `ILogger`, `Level`, `LoggerAdapterAbstract` with its `{}` formatter, and the console, default (no-op) and `java.util.logging` adapters.
- [mixin](./mixin/EGM.md): the consumer annotations, `Mixins`, `MixinEnvironment`, and the injection, transformer, gen, refmap, extensibility, connect, struct and throwables subpackages.
- [obfuscation](./obfuscation/EGM.md): `RemapperChain` and the `IMapping` structs (`MappingField`, `MappingMethod`, `MappingFieldSrg`).
- [service](./service/EGM.md): `MixinService` (ServiceLoader-based host-service selection), `IMixinService`, `MixinServiceAbstract`, the provider interfaces, and the service throwables.
- [transformers](./transformers/EGM.md): `MixinClassReader` (class-version guard), `MixinClassWriter` (a `ClassInfo`-backed common superclass) and `TreeTransformer`.
- [util](./util/EGM.md): `Bytecode`, `Annotations`, `ClassSignature`, `Locals`, `ConstraintParser`, `Quantifier`, `VersionNumber`, `JavaVersion`, `PrettyPrinter` and others, with the `asm`, `logging`, `perf` and `throwables` subpackages.

## Invariants

- None declared in this package's own file [package-info.java:org.spongepowered.asm].
- Across the subpackages, a working `IMixinService` must be resolvable before most classes are initialised. `Mixins`, `MixinEnvironment`, `MixinBootstrap` and several helpers obtain a logger from `MixinService.getService()` in their static initialisers. See [service](./service/EGM.md) and [mixin](./mixin/EGM.md).

## Side Effects

- None in this package's own file. Subpackage side effects are documented in each child contract.

## Verification

- Run: `mise exec -- ./gradlew compileJava`
- Run: `mise exec -- ./gradlew test`. The `test` source set is declared in `build.gradle`, but `src/test` does not exist yet, so there are no tests today.
