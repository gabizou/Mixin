# modlauncher9: org.spongepowered.asm.launch

## Overview

This package holds the ModLauncher 9+ entry points: `MixinTransformationService` (an `ITransformationService`) and `MixinLaunchPlugin` (an `ILaunchPluginService`). Only the `provides` clauses in `src/modularity/java/module-info.java` register them, because this source set has no `resources/` folder. They extend the shared `MixinTransformationServiceAbstract` and `MixinLaunchPluginLegacy` in `src/modlauncher`. They add the ModLauncher 9+ callback signatures and a synthetic module jar for Mixin-generated classes. The `platform` child package holds the SecureJar-aware root container.

## Interface

- `public class MixinLaunchPlugin extends MixinLaunchPluginLegacy`, which inherits `name()` returning `"mixin"` [MixinLaunchPlugin.java:MixinLaunchPlugin].
  - `@Override public void initializeLaunch(ITransformerLoader, NamedPath[] specialPaths)` calls the inherited `initializeLaunch(ITransformerLoader)`. `specialPaths` is unused [MixinLaunchPlugin.java:initializeLaunch].
- `public class MixinTransformationService extends MixinTransformationServiceAbstract`, which inherits `name()` returning `"mixin"` [MixinTransformationService.java:MixinTransformationService].
  - `@Override public List<Resource> completeScan(IModuleLayerManager layerManager)` [MixinTransformationService.java:completeScan].
  - The constant `VIRTUAL_JAR_CLASS = "cpw.mods.jarhandling.VirtualJar"` is private [MixinTransformationService.java:VIRTUAL_JAR_CLASS].
- Child package: [platform](./platform/EGM.md) groups `container`, which holds `ContainerHandleModLauncherEx`.

## Invariants

- `MixinLaunchPlugin` passes the inherited `initialize` check that the `"mixin"` launch plugin is a `MixinLaunchPluginLegacy`, because it is a subclass. The rest of the boot order is the same as the shared one in src/modlauncher/java/org/spongepowered/asm/launch/ [MixinLaunchPlugin.java:MixinLaunchPlugin].
- `completeScan` does the following [MixinTransformationService.java:completeScan], [MixinTransformationService.java:createVirtualJar], [MixinTransformationService.java:createShim]:
  1. It resolves this class's code source to a `Path`.
  2. If `MixinService.getService().getClassProvider().findClass("cpw.mods.jarhandling.VirtualJar", false)` succeeds, it returns one `Resource(IModuleLayerManager.Layer.GAME, [new VirtualJar("mixin_synthetic", codeSource, "org.spongepowered.asm.synthetic", "org.spongepowered.asm.synthetic.args")])`.
  3. Otherwise it returns one GAME-layer `Resource`, a `SecureJar` built with `JarMetadata.fromFileName(codeSource.resolve("mixin_synthetic"), {those two packages}, [])`.
- `URISyntaxException` is wrapped in `RuntimeException`. Any `Throwable` in `completeScan` is printed to stderr, and the method then returns `super.completeScan(layerManager)`, the ModLauncher default [MixinTransformationService.java:completeScan].
- `detectVirtualJar` treats only `ClassNotFoundException` as "not supported". It ignores its `layerManager` argument [MixinTransformationService.java:detectVirtualJar].
- Neither class overrides `beginScanning`. `runScan`, `initialize`, `arguments` and `transformers` are inherited [MixinTransformationService.java:MixinTransformationService].

## Side Effects

- `completeScan` contributes a synthetic module named `mixin_synthetic` to ModLauncher's GAME layer. It holds the packages `org.spongepowered.asm.synthetic` and `org.spongepowered.asm.synthetic.args` [MixinTransformationService.java:completeScan].
- It looks up `cpw.mods.jarhandling.VirtualJar` through the Mixin class provider without initializing it [MixinTransformationService.java:detectVirtualJar].

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileModlauncher9Java`. This uses a Java 16 toolchain, with the `main` and `modlauncher` outputs on the compile class path (build.gradle lines 140-144), against `cpw.mods:modlauncher:10.0.9` and `cpw.mods:securejarhandler:2.1.24` (build.gradle lines 205-208).
