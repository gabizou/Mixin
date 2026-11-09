# org.spongepowered.asm.mixin.connect

## Overview

This package defines `IMixinConnector`. Its javadoc calls a connector "a bootstrap agent for a mixin consumer" that calls back into `Mixins` to register its resources [IMixinConnector.java:IMixinConnector]. The package only defines the contract. The code that discovers, creates and calls connectors is `MixinConnectorManager` in src/main/java/org/spongepowered/asm/launch/platform. Connector class names come from the `MixinConnector` jar manifest attribute, which `MixinPlatformAgentDefault` reads in the same package.

## Interface

- `public interface IMixinConnector` with one method, `void connect()` [IMixinConnector.java:connect]
- `package-info.java` contains only package javadoc.

## Invariants

- An implementing class needs a no-argument constructor. `MixinConnectorManager.loadConnectors` creates instances with `getDeclaredConstructor().newInstance()` and does not call `setAccessible` (see src/main/java/org/spongepowered/asm/launch/platform/MixinConnectorManager.java).
- The class must be visible through `MixinService.getService().getClassProvider().findClass(name)`, and it must implement `IMixinConnector`. Otherwise `MixinConnectorManager` logs an error and skips it (same file).

## Side Effects

- None in this package. Whatever `connect()` does is up to the implementor.

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileJava`
- Run from the repo root: `mise exec -- ./gradlew test`
- `build.gradle` declares a `test` source set, but `src/test` does not exist yet. `test` therefore runs no tests today.
