# org.spongepowered.asm.launch.platform

## Overview

This package is the platform layer that `MixinBootstrap` (in the parent package) drives:

- `MixinPlatformManager` keeps an ordered map of mixin containers. It prepares and injects them, and it registers configs, token providers and connectors.
- Each `MixinContainer` creates one instance of every platform agent class for its container handle.
- Agents implement `IMixinPlatformAgent`. They decide whether to accept a container and read its manifest attributes. `MixinPlatformAgentDefault` handles the generic attributes.
- `MainAttributes` caches jar and directory manifests.
- `CommandLineOptions` collects the configs passed with `--mixin` or `--mixin.config`.
- `MixinConnectorManager` loads `IMixinConnector` classes and calls them at inject time.

Launcher integrations add agents through `IMixinService.getPlatformAgents()`:

- src/launchwrapper/java/org/spongepowered/asm/service/mojang/MixinServiceLaunchWrapper.java returns `MixinPlatformAgentFMLLegacy` and `MixinPlatformAgentLiteLoaderLegacy`.
- src/modlauncher/java/org/spongepowered/asm/service/modlauncher/MixinServiceModLauncher.java returns `MixinPlatformAgentMinecraftForge`.

All three agent classes extend `MixinPlatformAgentAbstract` and implement `IMixinPlatformServiceAgent`.

## Interface

- [container](./container/EGM.md): `IContainerHandle`, `ContainerHandleURI` and `ContainerHandleVirtual`.
- `public class MixinPlatformManager` [MixinPlatformManager.java:MixinPlatformManager]:
  - public methods: `MixinPlatformManager()`, `void init()`, `Collection<String> getPhaseProviderClasses()`, `final MixinContainer addContainer(IContainerHandle)`, `final void prepare(CommandLineOptions)`, `final void inject()` and `String getLaunchTarget()`
  - package-private methods: `setCompatibilityLevel(String)`, `addConfig(String, IMixinConfigSource)`, `addTokenProvider(String)` and `addConnector(String)`
- `public class MixinContainer` [MixinContainer.java:MixinContainer]:
  - `MixinContainer(MixinPlatformManager, IContainerHandle)`
  - `IContainerHandle getDescriptor()` and `Collection<String> getPhaseProviders()`
  - `void prepare()`, `void initPrimaryContainer()` and `void inject()`
- `public class MixinConnectorManager` [MixinConnectorManager.java:MixinConnectorManager]. Its constructor and all of its methods are package-private: `addConnector(String)`, `inject()`, `loadConnectors()` and `initConnectors()`.
- `public interface IMixinPlatformAgent` [IMixinPlatformAgent.java:IMixinPlatformAgent]:
  - `AcceptResult accept(MixinPlatformManager manager, IContainerHandle handle)`
  - `String getPhaseProvider()`
  - `void prepare()`, `void initPrimaryContainer()` and `void inject()`
  - nested `enum AcceptResult { ACCEPTED, REJECTED, INVALID }` [IMixinPlatformAgent.java:AcceptResult]
- `public interface IMixinPlatformServiceAgent extends IMixinPlatformAgent` [IMixinPlatformServiceAgent.java:IMixinPlatformServiceAgent]:
  - `void init()`, `String getSideName()` and `Collection<IContainerHandle> getMixinContainers()`
  - `@Deprecated void wire(Phase, IConsumer<Phase>)` and `@Deprecated void unwire()`
- `public abstract class MixinPlatformAgentAbstract implements IMixinPlatformAgent` [MixinPlatformAgentAbstract.java:MixinPlatformAgentAbstract]:
  - protected fields `manager` and `handle`, and a `protected static final ILogger logger`
  - `protected static String invokeStringMethod(ClassLoader, String className, String methodName)`
  - `@Deprecated wire(Phase, IConsumer<Phase>)` and `unwire()`, which do nothing
- `public class MixinPlatformAgentDefault extends MixinPlatformAgentAbstract`, which overrides only `prepare()` [MixinPlatformAgentDefault.java:prepare]
- `public final class MainAttributes` [MainAttributes.java:MainAttributes]:
  - `static MainAttributes of(File jar)` and `static MainAttributes of(URI uri)`
  - `final String get(String name)` and `final String get(Attributes.Name name)`
- `public final class CommandLineOptions` [CommandLineOptions.java:CommandLineOptions]:
  - `List<String> getConfigs()`
  - `static CommandLineOptions defaultArgs()`, `static CommandLineOptions ofArgs(List<String> args)` and `static CommandLineOptions of(List<String> configs)`

## Invariants

### Agent lifecycle

- The javadoc defines the order: `accept` runs per container, `prepare` runs during pre-init, `initPrimaryContainer` runs only on the primary container, and `inject` runs from the parent tweaker's inject. `AcceptResult.INVALID` means "no further containers should be offered" [IMixinPlatformAgent.java:IMixinPlatformAgent].
- The static `MixinContainer.agentClasses` is an `ArrayList`. It holds every `MixinService.getService().getPlatformAgents()` entry in order, followed by `org.spongepowered.asm.launch.platform.MixinPlatformAgentDefault`. The same live list is stored under `GlobalProperties.Keys.AGENTS` before it is filled [MixinContainer.java:agentClasses].
- For each agent class name, the `MixinContainer` constructor calls `Class.forName(name)` (one-argument form), then `getDeclaredConstructor().newInstance()`, then `agent.accept(manager, handle)` [MixinContainer.java:MixinContainer]. The result decides what happens next:
  - `ACCEPTED` keeps the agent.
  - `REJECTED` drops it for this container only.
  - `INVALID` removes the class name from the static list, so later containers and the `AGENTS` global no longer include it.
- Errors in the `MixinContainer` constructor [MixinContainer.java:MixinContainer]:
  - An `InstantiationException` rethrows its cause if that cause is a `RuntimeException`. Otherwise it is wrapped in `new RuntimeException(cause)`.
  - Any other `ReflectiveOperationException` goes to `logger.catching` and the agent is skipped.
  - Exceptions thrown by `accept()` are not caught.
- `MixinPlatformAgentAbstract.accept` stores `manager` and `handle` and returns `ACCEPTED`. `getPhaseProvider()` returns null, and `prepare`, `initPrimaryContainer` and `inject` do nothing. `toString()` returns `PlatformAgent[<SimpleClassName>:<handle>]` [MixinPlatformAgentAbstract.java:accept] [MixinPlatformAgentAbstract.java:toString].
- `invokeStringMethod` loads the class without initializing it (`Class.forName(name, false, loader)`) and calls a static no-argument method. It returns the `name()` of the `Enum` that method returns, or null on any exception [MixinPlatformAgentAbstract.java:invokeStringMethod].

### Default agent and manifest attributes

`MixinPlatformAgentDefault.prepare()` reads these handle attributes in this order [MixinPlatformAgentDefault.java:prepare]:

1. `MixinCompatibilityLevel` goes to `manager.setCompatibilityLevel`.
2. `MixinConfigs` is split on `,`, and each trimmed entry goes to `manager.addConfig(config, handle)`.
3. `MixinTokenProviders` is split on `,`, and each trimmed entry goes to `manager.addTokenProvider`.
4. `MixinConnector` is trimmed and goes to `manager.addConnector`.

The attribute names are constants in src/main/java/org/spongepowered/asm/util/Constants.java (`ManifestAttributes`).

### Manager handling of configs, compatibility levels and token providers

- `addConfig`:
  - A name ending in `.json` goes to `Mixins.addConfiguration(config, source)`.
  - A name containing `.json@` throws `MixinError("Setting config phase via manifest is no longer supported: <config>. Specify target in config instead")`.
  - Any other name is ignored silently [MixinPlatformManager.java:addConfig].
- `setCompatibilityLevel` calls `CompatibilityLevel.valueOf(level.toUpperCase(Locale.ROOT))`. An unknown level is logged as a warning and does not throw [MixinPlatformManager.java:setCompatibilityLevel].
- `addTokenProvider("cls@PHASE")` registers the class on that phase's environment, and is ignored silently if `Phase.forName` returns null. A name without `@` registers on the default environment [MixinPlatformManager.java:addTokenProvider].

### Manager container lifecycle

- `containers` is a `LinkedHashMap<IContainerHandle, MixinContainer>`, so insertion order sets the order of `prepare` and `inject` [MixinPlatformManager.java:containers].
- `init()` adds `MixinService.getService().getPrimaryContainer()` first and then calls `scanForContainers()`. It has no guard against repeated calls [MixinPlatformManager.java:init].
- `addContainer` returns the existing `MixinContainer` when the handle is already a key. Otherwise it creates a container, then adds any nested handles that are not yet present, parent first [MixinPlatformManager.java:addContainer] [MixinPlatformManager.java:addNestedContainers].
- A container created after `prepare()` has run is prepared immediately [MixinPlatformManager.java:createContainerFor].
- `prepare(args)` sets `prepared = true`, calls `prepare()` on every container, then calls `addConfig(config, null)` for each `args.getConfigs()` entry. It has no guard against repeated calls [MixinPlatformManager.java:prepare].
- `inject()` is guarded by `injected`, so a second call does nothing. It runs these steps in order [MixinPlatformManager.java:inject]:
  1. `primaryContainer.initPrimaryContainer()`, if the primary container is non-null
  2. `scanForContainers()`
  3. `inject()` on each container. An exception is printed and the loop continues.
  4. `connectors.inject()`
- `scanForContainers` turns an `AbstractMethodError` from `getMixinContainers()` into `ServiceVersionError("Mixin service is out of date")`. An exception from `addContainer` for one handle is printed and that handle is skipped [MixinPlatformManager.java:scanForContainers].
- `getPhaseProviderClasses()` throws `NullPointerException` before `init()`, because `primaryContainer` is null until then [MixinPlatformManager.java:getPhaseProviderClasses].
- `getLaunchTarget()` always returns `net.minecraft.client.main.Main` [MixinPlatformManager.java:getLaunchTarget].

### Connectors

- `MixinConnectorManager` keeps connector names in a `LinkedHashSet`, so names are de-duplicated and kept in insertion order [MixinConnectorManager.java:connectorClasses].
- `loadConnectors` resolves each name with the service `IClassProvider.findClass(name)` [MixinConnectorManager.java:loadConnectors]:
  - A class that is not an `IMixinConnector` is logged as an error and skipped.
  - A `ClassNotFoundException` goes to `logger.catching` and the name is skipped.
  - Instantiation uses `getDeclaredConstructor().newInstance()`. A reflection failure is logged as a warning and the name is skipped.
  - After the loop, the name set is cleared.
- `initConnectors` calls `connect()` on every loaded connector in load order. An exception is logged as a warning and the loop continues. The loaded list is never cleared [MixinConnectorManager.java:initConnectors].

### MainAttributes

- The static `instances` map caches one `MainAttributes` per `URI`, including a null URI. Entries are never evicted. Cached attributes are not re-read if the file changes [MainAttributes.java:instances] [MainAttributes.java:of].
- How attributes are read [MainAttributes.java:getAttributes]:
  - A `file:` URI that points to a file is read with `JarFile.getManifest()`. One that points to a directory is read from `META-INF/MANIFEST.MF`.
  - Any other scheme is read through NIO `Paths.get(uri)`, and only on Java 7 or later.
  - If nothing is found, the result is an empty `Attributes`.
  - A null URI gives null attributes, and `get` then returns null.
- `IOException` is swallowed. `FileSystemNotFoundException` and `InvalidPathException` print a stack trace and the empty-attributes fallback is used [MainAttributes.java:getJarAttributes] [MainAttributes.java:getNioAttributes].

### CommandLineOptions

- `parseArgs` adds the argument that comes right after each `--mixin` or `--mixin.config` token. A trailing flag with no value adds nothing [CommandLineOptions.java:parseArgs].
- `ofArgs(null)` and `defaultArgs()` parse the system property `sun.java.command`, split on single spaces. When that property is unset, the config list is empty [CommandLineOptions.java:ofArgs].
- `of(configs)` copies the list without parsing it. `getConfigs()` returns an unmodifiable view [CommandLineOptions.java:of] [CommandLineOptions.java:getConfigs].

## Side Effects

- Static state:
  - `MixinContainer.agentClasses` is shared through the `mixin.agents` global property and can lose entries when an agent returns `INVALID` [MixinContainer.java:agentClasses].
  - `MainAttributes.instances` is an unsynchronized `HashMap` [MainAttributes.java:instances].
  - The static loggers in `MixinPlatformManager`, `MixinContainer`, `MixinConnectorManager` and `MixinPlatformAgentAbstract` each call `MixinService.getService().getLogger("mixin")` during class initialization [MixinPlatformAgentAbstract.java:logger].
- Global state outside this package:
  - `Mixins.addConfiguration` [MixinPlatformManager.java:addConfig]
  - `MixinEnvironment.setCompatibilityLevel` [MixinPlatformManager.java:setCompatibilityLevel]
  - `MixinEnvironment.registerTokenProviderClass` [MixinPlatformManager.java:addTokenProvider]
- Reflection:
  - Agent and connector classes are loaded and created by reflection, which runs their static initializers [MixinContainer.java:MixinContainer] [MixinConnectorManager.java:loadConnectors].
- I/O and system properties:
  - `MainAttributes` opens jar files and manifest files [MainAttributes.java:getJarAttributes] [MainAttributes.java:getDirAttributes].
  - `CommandLineOptions.ofArgs(null)` reads `System.getProperty("sun.java.command")` [CommandLineOptions.java:ofArgs].
- Console output:
  - `printStackTrace()` writes to stderr for failures in container inject and container scan [MixinPlatformManager.java:inject] [MixinPlatformManager.java:scanForContainers].
  - It also writes NIO manifest failures [MainAttributes.java:getNioAttributes].

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileJava`
- Run from the repo root: `mise exec -- ./gradlew test`
- `build.gradle` declares a `test` source set, but `src/test` does not exist yet. `test` therefore runs no tests today.
