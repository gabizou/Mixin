# modlauncher4: org.spongepowered.asm.launch

## Overview

This package holds `MixinTransformationServiceLegacy`, the `ITransformationService` for ModLauncher 4-8. `src/modlauncher4/resources/META-INF/services/cpw.mods.modlauncher.api.ITransformationService` registers it. It adds the `beginScanning` callback that those ModLauncher versions require, and inherits everything else from `MixinTransformationServiceAbstract` in `src/modlauncher`.

## Interface

- `public class MixinTransformationServiceLegacy extends MixinTransformationServiceAbstract` [MixinTransformationServiceLegacy.java:MixinTransformationServiceLegacy]
  - `@Override public void beginScanning(IEnvironment environment)` has an empty body [MixinTransformationServiceLegacy.java:beginScanning].
  - It inherits unchanged `name()` (`"mixin"`), `arguments`, `argumentValues`, `onLoad`, `initialize`, `runScan` and `transformers` from src/modlauncher/java/org/spongepowered/asm/launch/MixinTransformationServiceAbstract.java.

## Invariants

- `beginScanning` does nothing. The inherited `initialize` still requires a launch plugin named `"mixin"` that is a `MixinLaunchPluginLegacy`, and throws `MixinInitialisationError` otherwise [MixinTransformationServiceLegacy.java:beginScanning].

## Side Effects

- None

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileModlauncher4Java`. This uses a Java 8 toolchain with source and target `1.8`, with the `main` and `modlauncher` outputs on the compile class path (build.gradle lines 134-139), against `cpw.mods:modlauncher:7.0.0` (build.gradle lines 197-203).
