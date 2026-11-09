# launchwrapper: org.spongepowered.asm.mixin.transformer

## Overview

This package holds `Proxy`, the LegacyLauncher `IClassTransformer` that `MixinServiceLaunchWrapper.beginPhase()` registers with `Launch.classLoader`. Every proxy instance shares one Mixin transformer, and only the most recently constructed proxy forwards class bytes to it. Re-registering the proxy at each phase therefore moves Mixin to the end of the LaunchWrapper transformer chain without applying mixins twice.

## Interface

- `public final class Proxy implements net.minecraft.launchwrapper.IClassTransformer, org.spongepowered.asm.service.ILegacyClassTransformer` [Proxy.java:Proxy]
  - `public Proxy()` [Proxy.java:Proxy()]
  - `byte[] transform(String name, String transformedName, byte[] basicClass)` (LegacyLauncher entry point) [Proxy.java:transform]
  - `byte[] transformClassBytes(String name, String transformedName, byte[] basicClass)` [Proxy.java:transformClassBytes]
  - `String getName()` returns `"org.spongepowered.asm.mixin.transformer.Proxy"` (`getClass().getName()`) [Proxy.java:getName]
  - `boolean isDelegationExcluded()` always returns `true`, so `MixinServiceLaunchWrapper` never adds the proxy to its own delegation list [Proxy.java:isDelegationExcluded]

## Invariants

- All instances share the single static `MixinTransformer` (the same-package `org.spongepowered.asm.mixin.transformer.MixinTransformer` from the main source set). It is created when the `Proxy` class initializes [Proxy.java:transformer].
- Each constructor call sets `isActive = false` on every earlier proxy, then appends itself to the static `proxies` list. Only the newest proxy is active [Proxy.java:Proxy()], [Proxy.java:proxies].
- An active proxy returns `transformer.transformClassBytes(name, transformedName, basicClass)`. An inactive proxy returns the same `basicClass` reference unchanged [Proxy.java:transform], [Proxy.java:transformClassBytes].
- The `proxies` list is never cleared and is not synchronized. No exceptions are caught, so transformer failures reach LaunchWrapper [Proxy.java:proxies].

## Side Effects

- Class initialization constructs one `MixinTransformer` [Proxy.java:transformer].
- The constructor logs at debug `"Adding new mixin transformer proxy #{}"` (the 1-based count) through `MixinService.getService().getLogger("mixin")` [Proxy.java:Proxy()].

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileLaunchwrapperJava`. This uses a Java 8 toolchain with source and target `1.8` (build.gradle lines 124-128) against `net.minecraft:launchwrapper:1.11` (build.gradle lines 181-185).
