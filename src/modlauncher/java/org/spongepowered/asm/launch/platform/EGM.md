# modlauncher: org.spongepowered.asm.launch.platform

## Overview

This package holds `MixinPlatformAgentMinecraftForge`, the only platform agent that `MixinServiceModLauncher.getPlatformAgents()` lists. It accepts no containers. Its only job is to report the game side from ModLauncher's launch target. The `container` child package holds the ModLauncher root container handle.

## Interface

- `public class MixinPlatformAgentMinecraftForge extends MixinPlatformAgentAbstract implements IMixinPlatformServiceAgent` [MixinPlatformAgentMinecraftForge.java:MixinPlatformAgentMinecraftForge]
  - `void init()` does nothing [MixinPlatformAgentMinecraftForge.java:init].
  - `AcceptResult accept(MixinPlatformManager, IContainerHandle)` always returns `AcceptResult.REJECTED` [MixinPlatformAgentMinecraftForge.java:accept].
  - `String getSideName()` [MixinPlatformAgentMinecraftForge.java:getSideName].
  - `Collection<IContainerHandle> getMixinContainers()` returns `null` [MixinPlatformAgentMinecraftForge.java:getMixinContainers].
- Child package: [container](./container/EGM.md) holds `ContainerHandleModLauncher`.

## Invariants

- `getSideName()` decides the side as follows [MixinPlatformAgentMinecraftForge.java:getSideName]:
  1. It reads `Launcher.INSTANCE.environment().getProperty(IEnvironment.Keys.LAUNCHTARGET.get())`, defaults to `"missing"`, and lower-cases it.
  2. It returns `Constants.SIDE_SERVER` if that string contains `server`, else `Constants.SIDE_CLIENT` if it contains `client`.
  3. Otherwise it finds the launch handler for that target and calls its `getDist` method reflectively. The method must be declared on the concrete handler class. It applies the same server-then-client test to the result.
  4. If no handler exists, nothing matches, or any exception occurs, it returns `null`.
- The agent does no CoreMod or manifest-attribute handling, because it rejects every container [MixinPlatformAgentMinecraftForge.java:accept].

## Side Effects

- It reads the global `Launcher.INSTANCE` environment and reflectively calls `getDist` on the launch handler [MixinPlatformAgentMinecraftForge.java:getSideName].

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileModlauncherJava`. This uses a Java 8 toolchain with source and target `1.8` (build.gradle lines 129-133) against `cpw.mods:modlauncher:7.0.0` (build.gradle lines 187-195).
