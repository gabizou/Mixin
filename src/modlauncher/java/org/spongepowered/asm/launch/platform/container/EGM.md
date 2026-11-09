# modlauncher: org.spongepowered.asm.launch.platform.container

## Overview

This package holds `ContainerHandleModLauncher`, the root container that `MixinServiceModLauncher.getPrimaryContainer()` returns when the ModLauncher API specification version is below `8.0`. ModLauncher passes resources to the Mixin launch plugin, and this container collects them as child container handles. The ModLauncher 9+ subclass `ContainerHandleModLauncherEx` lives in the `modlauncher9` source set.

## Interface

- `public class ContainerHandleModLauncher extends ContainerHandleVirtual` [ContainerHandleModLauncher.java:ContainerHandleModLauncher]
  - `public ContainerHandleModLauncher(String name)`. The service builds it reflectively with the name `"ModLauncher"` [ContainerHandleModLauncher.java:ContainerHandleModLauncher(String)].
  - `void addResource(String name, Path path)` and `void addResource(Entry<String, Path> entry)` each add a new `Resource` child [ContainerHandleModLauncher.java:addResource(String,Path)], [ContainerHandleModLauncher.java:addResource(Entry)].
  - `void addResource(Object resource)` accepts a `Map.Entry`, treated as `Entry<String, Path>` [ContainerHandleModLauncher.java:addResource(Object)].
  - `void addResources(List<?> resources)` calls `addResource(Object)` for each element [ContainerHandleModLauncher.java:addResources].
  - `toString()` returns `"ModLauncher Root Container(<name>:<hex hashCode>)"` [ContainerHandleModLauncher.java:toString].
- `class Resource extends ContainerHandleURI` is a package-private, non-static inner class. Its constructor is `Resource(String name, Path path)`, which passes `path.toUri()` to the superclass [ContainerHandleModLauncher.java:Resource].
  - `getId()` returns the name without the text after the last `.`, when that `.` is not the first character [ContainerHandleModLauncher.java:Resource.getId].
  - `getDescription()` returns the absolute path [ContainerHandleModLauncher.java:Resource.getDescription].
  - `getName()` and `getPath()` return the name and path [ContainerHandleModLauncher.java:Resource.getName].
  - `toString()` returns `"ContainerHandleModLauncher.Resource(<name>:<path>)"` [ContainerHandleModLauncher.java:Resource.toString].

## Invariants

- `addResource(Object)` adds only `Map.Entry` values. Any other type is logged as an error and dropped. A `null` resource throws `NullPointerException` from `resource.getClass()` [ContainerHandleModLauncher.java:addResource(Object)].
- `addResources` dispatches to `addResource(Object)` virtually, so the `SecureJar` handling in the ModLauncher 9 subclass applies [ContainerHandleModLauncher.java:addResources].
- `getAttribute` and the handling of nested containers come from core `ContainerHandleVirtual` and `ContainerHandleURI` (src/main/java/org/spongepowered/asm/launch/platform/container/). This file does not override them [ContainerHandleModLauncher.java:ContainerHandleModLauncher].

## Side Effects

- For an unrecognised type, it logs `"Unrecognised resource type {} passed to {}"` through `MixinService.getService().getLogger("mixin")` [ContainerHandleModLauncher.java:addResource(Object)].
- It appends child handles to the inherited `ContainerHandleVirtual` list [ContainerHandleModLauncher.java:addResource(String,Path)].

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileModlauncherJava`. This uses a Java 8 toolchain with source and target `1.8` (build.gradle lines 129-133) against `cpw.mods:modlauncher:7.0.0`, the `legacyModlauncherVersion` in gradle.properties (build.gradle lines 187-195).
