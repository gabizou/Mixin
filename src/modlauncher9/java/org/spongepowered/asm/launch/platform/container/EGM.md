# modlauncher9: org.spongepowered.asm.launch.platform.container

## Overview

This package holds `ContainerHandleModLauncherEx`, the root container for ModLauncher 9+. `MixinServiceModLauncher` creates it reflectively, by class name, when the ModLauncher API specification version is at least `8.0`. It extends the shared `ContainerHandleModLauncher` in `src/modlauncher` so that it also accepts `cpw.mods.jarhandling.SecureJar` resources.

## Interface

- `public class ContainerHandleModLauncherEx extends ContainerHandleModLauncher`. Its constructor is `public ContainerHandleModLauncherEx(String name)` [ContainerHandleModLauncherEx.java:ContainerHandleModLauncherEx].
  - `@Override public void addResource(Object resource)` [ContainerHandleModLauncherEx.java:addResource(Object)].
- `static class SecureJarResource extends ContainerHandleURI` is package-private. Its constructor is `SecureJarResource(SecureJar)`, which passes `jar.getPrimaryPath().toUri()` to the superclass [ContainerHandleModLauncherEx.java:SecureJarResource].
  - `getId()` returns `jar.name()` without the text after the last `.`, when that `.` is not the first character [ContainerHandleModLauncherEx.java:SecureJarResource.getId].
  - `getDescription()` returns the absolute path of `jar.getRootPath()` [ContainerHandleModLauncherEx.java:SecureJarResource.getDescription].
  - `getName()` returns `jar.name()`, and `getPath()` returns `jar.getPrimaryPath()` [ContainerHandleModLauncherEx.java:SecureJarResource.getName].
  - `toString()` returns `"SecureJarResource(<name>)"` [ContainerHandleModLauncherEx.java:SecureJarResource.toString].

## Invariants

- `addResource(Object)` wraps a `SecureJar` in a `SecureJarResource` child. Every other type goes to `super.addResource(Object)`, which accepts a `Map.Entry<String, Path>` and logs an error for anything else [ContainerHandleModLauncherEx.java:addResource(Object)].
- The inherited `addResources(List<?>)` dispatches to this override for each element [ContainerHandleModLauncherEx.java:addResource(Object)].

## Side Effects

- It appends `SecureJarResource` children to the inherited `ContainerHandleVirtual` list [ContainerHandleModLauncherEx.java:addResource(Object)].

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileModlauncher9Java`. This uses a Java 16 toolchain, with the `main` and `modlauncher` outputs on the compile class path (build.gradle lines 140-144), against `cpw.mods:modlauncher:10.0.9` (the `modlauncherVersion` in gradle.properties) and `cpw.mods:securejarhandler:2.1.24` (build.gradle lines 205-208).
