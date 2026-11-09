# org.spongepowered.asm.launch.platform.container

## Overview

This package holds the platform-agnostic container handles. A container handle is a source of mixin configs, such as a jar, a directory or a virtual grouping, plus its manifest-style attributes and any nested containers. `MixinPlatformManager` in the parent package keys its containers by these handles, and platform agents read their attributes.

Launcher integrations subclass these types:

- src/modlauncher/java/org/spongepowered/asm/launch/platform/container/ContainerHandleModLauncher.java extends `ContainerHandleVirtual`. Its nested `Resource` class extends `ContainerHandleURI`.
- src/modlauncher9/java/org/spongepowered/asm/launch/platform/container/ContainerHandleModLauncherEx.java has a nested `SecureJarResource` that extends `ContainerHandleURI`.

## Interface

- `public interface IContainerHandle extends IMixinConfigSource` [IContainerHandle.java:IContainerHandle]. `IMixinConfigSource` is in src/main/java/org/spongepowered/asm/mixin/extensibility.
  - `String getAttribute(String name)`, which returns null when the attribute is absent [IContainerHandle.java:getAttribute]
  - `Collection<IContainerHandle> getNestedContainers()` [IContainerHandle.java:getNestedContainers]
- `public class ContainerHandleURI implements IContainerHandle` [ContainerHandleURI.java:ContainerHandleURI]:
  - `public ContainerHandleURI(URI uri)`
  - `String getId()`, `String getDescription()`, `URI getURI()` and `@Deprecated File getFile()`
  - `getAttribute(String)`, `getNestedContainers()`, `equals`, `hashCode` and `toString`
- `public class ContainerHandleVirtual implements IContainerHandle` [ContainerHandleVirtual.java:ContainerHandleVirtual]:
  - `public ContainerHandleVirtual(String name)`
  - `String getId()`, `String getDescription()` and `String getName()`
  - fluent `ContainerHandleVirtual setAttribute(String key, String value)` and `ContainerHandleVirtual add(IContainerHandle nested)`
  - `getAttribute(String)`, `getNestedContainers()`, `equals`, `hashCode` and `toString`

## Invariants

### ContainerHandleURI

- `getId()` always returns null [ContainerHandleURI.java:getId].
- `getDescription()` returns `uri.toString()`. `toString()` returns `ContainerHandleURI(<uri>)` [ContainerHandleURI.java:getDescription] [ContainerHandleURI.java:toString].
- The constructor reads attributes once with `MainAttributes.of(uri)`, and `getAttribute(name)` delegates to that object [ContainerHandleURI.java:ContainerHandleURI] [ContainerHandleURI.java:getAttribute]. `MainAttributes` is in the parent package `launch/platform`.
- `getNestedContainers()` always returns the immutable `Collections.emptyList()` [ContainerHandleURI.java:getNestedContainers].
- `getFile()` returns `Files.toFile(uri)` only when the URI is non-null and its scheme is `file`. Otherwise it returns null [ContainerHandleURI.java:getFile].
- `equals` is true only for another `ContainerHandleURI` with an equal `uri`, and `hashCode()` is `uri.hashCode()`. With a null URI, `equals`, `hashCode` and `getDescription` throw `NullPointerException` [ContainerHandleURI.java:equals] [ContainerHandleURI.java:hashCode].

### ContainerHandleVirtual

- `equals(obj)` is true only when `obj` is a `String` equal to `name`. It is false for every `ContainerHandleVirtual`, including the same instance, so `x.equals(x)` is false. Hash-based collections still find the same instance because they compare references before calling `equals`. Two virtual handles with the same name are therefore separate keys in `MixinPlatformManager.containers` [ContainerHandleVirtual.java:equals].
- `hashCode()` is `name.hashCode()`, so a null name makes `hashCode`, `toString` and `getDescription` throw `NullPointerException` [ContainerHandleVirtual.java:hashCode].
- `toString()` returns `ContainerHandleVirtual(<name>:<hashCode in hex>)` and `getDescription()` returns `toString()`. Both `getId()` and `getName()` return `name` [ContainerHandleVirtual.java:toString] [ContainerHandleVirtual.java:getId].
- Attributes live in a `HashMap`. `setAttribute` overwrites an existing key, and a missing key reads as null. `setAttribute` and `add` both return `this` [ContainerHandleVirtual.java:setAttribute] [ContainerHandleVirtual.java:getAttribute].
- Nested handles live in a `LinkedHashSet`, so they keep insertion order and are de-duplicated by the nested handle's own `equals`/`hashCode`. `getNestedContainers()` returns an unmodifiable live view of that set [ContainerHandleVirtual.java:add] [ContainerHandleVirtual.java:getNestedContainers].

## Side Effects

- The `ContainerHandleURI` constructor calls `MainAttributes.of(uri)`. That call fills the static `MainAttributes` cache and may open the jar or the manifest file [ContainerHandleURI.java:ContainerHandleURI].
- `ContainerHandleVirtual.setAttribute` and `add` change the instance's attribute map and nested set [ContainerHandleVirtual.java:setAttribute] [ContainerHandleVirtual.java:add].
- The package has no static state.

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileJava`
- Run from the repo root: `mise exec -- ./gradlew test`
- `build.gradle` declares a `test` source set, but `src/test` does not exist yet. `test` therefore runs no tests today.
