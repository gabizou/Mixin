# org.spongepowered.asm.service

## Overview

This package is the service layer that connects Mixin's core to a host environment (a launcher):

- `MixinService` is a static singleton. It finds one `IMixinService` and one `IGlobalPropertyService` with `java.util.ServiceLoader`. Before that, it runs every registered `IMixinServiceBootstrap`.
- `MixinServiceAbstract` is the base class that both launcher services extend.
- The remaining interfaces are optional and required service components: class lookup, bytecode, transformers, class tracking, audit trail, synthetic classes and the global property store. There are also three throwables for service failures.

### Service registrations

The core source set registers no providers. The launcher source sets do:

| Source of registration | `IMixinServiceBootstrap` | `IMixinService` | `IGlobalPropertyService` |
| --- | --- | --- | --- |
| src/launchwrapper/resources/META-INF/services | `org.spongepowered.asm.service.mojang.MixinServiceLaunchWrapperBootstrap` | `org.spongepowered.asm.service.mojang.MixinServiceLaunchWrapper` | `org.spongepowered.asm.service.mojang.Blackboard` |
| src/modlauncher4/resources/META-INF/services | `org.spongepowered.asm.service.modlauncher.MixinServiceModLauncherBootstrap` | `org.spongepowered.asm.service.modlauncher.MixinServiceModLauncher` | `org.spongepowered.asm.service.modlauncher.Blackboard` |
| src/modularity/java/module-info.java (`provides ... with`) | `org.spongepowered.asm.service.modlauncher.MixinServiceModLauncherBootstrap` | `org.spongepowered.asm.service.modlauncher.MixinServiceModLauncher` | `org.spongepowered.asm.service.modlauncher.Blackboard` |

The implementations live in src/launchwrapper/java/org/spongepowered/asm/service/mojang and src/modlauncher/java/org/spongepowered/asm/service/modlauncher:

- `MixinServiceLaunchWrapper` and `MixinServiceModLauncher` both extend `MixinServiceAbstract`. Both override `createLogger` to return their own `LoggerAdapterLog4j2`.
- `MixinServiceLaunchWrapper` also implements `IClassProvider`, `IClassBytecodeProvider` and `ITransformerProvider`.
- The ModLauncher side splits these out into `ModLauncherClassProvider` (`IClassProvider`), `ModLauncherClassTracker` (`IClassTracker`) and `ModLauncherAuditTrail` (`IMixinAuditTrail`).
- `LaunchClassLoaderUtil` (`IClassTracker`) and `LegacyTransformerHandle` (`ILegacyClassTransformer`) are on the LaunchWrapper side.
- Both `Blackboard` classes implement `IGlobalPropertyService` and have a nested `Key implements IPropertyKey`.

## Interface

- `public final class MixinService` [MixinService.java:MixinService]:
  - `static void boot()` [MixinService.java:boot]
  - `static IMixinService getService()` [MixinService.java:getService]
  - `static IGlobalPropertyService getGlobalPropertyService()` [MixinService.java:getGlobalPropertyService]
  - package-private nested `LogBuffer` [MixinService.java:LogBuffer]
- `public interface IMixinService` [IMixinService.java:IMixinService]:
  - Identity: `String getName()` and `boolean isValid()`
  - Lifecycle: `void prepare()`, `Phase getInitialPhase()`, `void offer(IMixinInternal)`, `void init()`, `void beginPhase()` and `void checkEnv(Object bootSource)`
  - Components: `ReEntranceLock getReEntranceLock()`, `IClassProvider getClassProvider()`, `IClassBytecodeProvider getBytecodeProvider()`, `ITransformerProvider getTransformerProvider()`, `IClassTracker getClassTracker()` and `IMixinAuditTrail getAuditTrail()`
  - Containers: `Collection<String> getPlatformAgents()`, `IContainerHandle getPrimaryContainer()` and `Collection<IContainerHandle> getMixinContainers()`
  - Environment: `InputStream getResourceAsStream(String name)`, `String getSideName()`, `CompatibilityLevel getMinCompatibilityLevel()` and `CompatibilityLevel getMaxCompatibilityLevel()`
  - Logging: `ILogger getLogger(String name)`
- `public abstract class MixinServiceAbstract implements IMixinService` [MixinServiceAbstract.java:MixinServiceAbstract]:
  - protected constants `LAUNCH_PACKAGE = "org.spongepowered.asm.launch."`, `MIXIN_PACKAGE = "org.spongepowered.asm.mixin."` and `SERVICE_PACKAGE = "org.spongepowered.asm.service."`
  - `protected final ReEntranceLock lock`
  - `protected final <T extends IMixinInternal> T getInternal(Class<T>)`
  - `protected final void getContainersFromAgents(Builder<IContainerHandle>)`
  - `protected ILogger createLogger(String)`
  - `@Deprecated wire(Phase, IConsumer<Phase>)` and `unwire()`
- `public interface IMixinServiceBootstrap`: `String getName()`, `String getServiceClassName()` and `void bootstrap()` [IMixinServiceBootstrap.java:IMixinServiceBootstrap]
- `public interface IGlobalPropertyService` [IGlobalPropertyService.java:IGlobalPropertyService]:
  - `IPropertyKey resolveKey(String name)`
  - `<T> T getProperty(IPropertyKey)` and `<T> T getProperty(IPropertyKey, T defaultValue)`
  - `void setProperty(IPropertyKey, Object)`
  - `String getPropertyString(IPropertyKey, String defaultValue)`
- `public interface IPropertyKey` is a marker interface [IPropertyKey.java:IPropertyKey]
- `public interface IClassProvider` [IClassProvider.java:IClassProvider]:
  - `@Deprecated URL[] getClassPath()`
  - `Class<?> findClass(String)` and `Class<?> findClass(String, boolean initialize)`
  - `Class<?> findAgentClass(String, boolean initialize)`
  - All `find*` methods throw `ClassNotFoundException`.
- `public interface IClassBytecodeProvider` has three overloads, each throwing `ClassNotFoundException` and `IOException` [IClassBytecodeProvider.java:IClassBytecodeProvider]:
  - `ClassNode getClassNode(String name)`
  - `ClassNode getClassNode(String name, boolean runTransformers)`
  - `ClassNode getClassNode(String name, boolean runTransformers, int readerFlags)`
- `public interface IClassTracker`: `void registerInvalidClass(String)`, `boolean isClassLoaded(String)` and `String getClassRestrictions(String)` [IClassTracker.java:IClassTracker]
- `public interface ITransformerProvider`: `Collection<ITransformer> getTransformers()`, `Collection<ITransformer> getDelegatedTransformers()` and `void addTransformerExclusion(String)` [ITransformerProvider.java:ITransformerProvider]
- Transformer interfaces:
  - `public interface ITransformer`: `String getName()` and `boolean isDelegationExcluded()` [ITransformer.java:ITransformer]
  - `public interface ILegacyClassTransformer extends ITransformer`: `byte[] transformClassBytes(String name, String transformedName, byte[] basicClass)` [ILegacyClassTransformer.java:transformClassBytes]
  - `public interface ITreeClassTransformer extends ITransformer`: `boolean transformClassNode(String name, String transformedName, ClassNode)` [ITreeClassTransformer.java:transformClassNode]
- `public interface IMixinAuditTrail`: `onApply(String className, String mixinName)`, `onPostProcess(String className)` and `onGenerate(String className, String generatorName)` [IMixinAuditTrail.java:IMixinAuditTrail]
- Synthetic classes:
  - `public interface ISyntheticClassRegistry`: `ISyntheticClassInfo findSyntheticClass(String name)` [ISyntheticClassRegistry.java:findSyntheticClass]
  - `public interface ISyntheticClassInfo`: `IMixinInfo getMixin()`, `String getName()`, `String getClassName()` and `boolean isLoaded()` [ISyntheticClassInfo.java:ISyntheticClassInfo]
- `public interface IMixinInternal` is a marker for internal components offered to services [IMixinInternal.java:IMixinInternal]
- Throwables:
  - `public class ServiceInitialisationException extends RuntimeException`, with constructors `()`, `(String)`, `(Throwable)` and `(String, Throwable)` [ServiceInitialisationException.java:ServiceInitialisationException]
  - `public class ServiceNotAvailableError extends Error`, with constructor `(String)` [ServiceNotAvailableError.java:ServiceNotAvailableError]
  - `public class ServiceVersionError extends Error`, with constructor `(String)` [ServiceVersionError.java:ServiceVersionError]

## Invariants

### Discovery and selection (MixinService)

- The singleton `MixinService.instance` is created lazily by `getInstance()`, which is not synchronized. `boot()` only forces that creation. There is no reset method [MixinService.java:getInstance] [MixinService.java:boot].
- The constructor runs the bootstrap services. It iterates `ServiceLoader.load(IMixinServiceBootstrap.class, MixinService.class.getClassLoader())` and, for each provider, calls `bootstrap()` and then records `getServiceClassName()` in `bootedServices` [MixinService.java:runBootServices]. Failures do not stop the loop:
  - A `ServiceInitialisationException` is the expected "this host is not present" signal. It is debug-logged as `Mixin bootstrap service {} is not available: {}`.
  - Any other `Throwable`, including a `ServiceConfigurationError` thrown by `iter.next()`, is debug-logged as `Catching {}:{} initialising service`.
- `getService()` is synchronized. The first call iterates `ServiceLoader.load(IMixinService.class, <same class loader>)` and returns the first provider whose `isValid()` is true, in `ServiceLoader` order [MixinService.java:getServiceInstance] [MixinService.java:initService]. Providers that are not selected are recorded:
  - An invalid provider is recorded as `INVALID[<getName()>]`.
  - A `ServiceConfigurationError` is only counted.
  - Any other `Throwable` is recorded as `ERROR[<simple class name of the top stack frame>]` and debug-logged.
- When no provider is valid, `getService()` throws `ServiceNotAvailableError("No mixin host service is available. Services: <entries joined by ', '>")`. If any `ServiceConfigurationError`s were counted, ` and N other invalid services.` is appended [MixinService.java:initService].
- There is no hard-coded fallback `IMixinService` and no system-property override. The only fallback is for logging [MixinService.java:getServiceInstance]:
  - On any `Error`, buffered messages are flushed to `new LoggerAdapterConsole("mixin").setDebugStream(System.err)`, the error is logged at `ERROR`, and the error is rethrown.
  - `service` stays null, so the next `getService()` repeats discovery. Bootstrap services are not run again.
- After selection, the service is cached for the life of the singleton. The buffered startup log is flushed to `service.getLogger("mixin")`, and `LogBuffer` then forwards directly to that logger [MixinService.java:LogBuffer.flush].
- `getGlobalPropertyService()` is lazy and not synchronized. It iterates `ServiceLoader.load(IGlobalPropertyService.class, <same class loader>)` and returns the first provider that can be created, with no validity check. Any `ServiceConfigurationError` or `Throwable` is skipped silently. When none is found, it throws `ServiceNotAvailableError("No mixin global property service is available")` [MixinService.java:initPropertyService].

### Contract rules (IMixinService and components)

- `getClassProvider()` and `getBytecodeProvider()` must not return null [IMixinService.java:getClassProvider] [IMixinService.java:getBytecodeProvider].
- `getTransformerProvider()`, `getClassTracker()` and `getAuditTrail()` may return null [IMixinService.java:getTransformerProvider] [IMixinService.java:getClassTracker] [IMixinService.java:getAuditTrail].
- `getLogger` must not return null and should be thread-safe [IMixinService.java:getLogger].
- `getResourceAsStream` returns null when the resource is missing [IMixinService.java:getResourceAsStream].
- `IClassTracker.getClassRestrictions` returns a comma-separated list, or `""` when nothing applies [IClassTracker.java:getClassRestrictions].
- `ISyntheticClassRegistry.findSyntheticClass` returns null for an unknown name [ISyntheticClassRegistry.java:findSyntheticClass].
- `ITransformer.isDelegationExcluded()` returning true excludes the transformer from delegation [ITransformer.java:isDelegationExcluded].
- `ITreeClassTransformer.transformClassNode` returns true when it changed the class [ITreeClassTransformer.java:transformClassNode].

### MixinServiceAbstract defaults

- Default return values [MixinServiceAbstract.java:getInitialPhase] [MixinServiceAbstract.java:getMinCompatibilityLevel]:
  - `getInitialPhase()` returns `Phase.PREINIT`.
  - `getMinCompatibilityLevel()` and `getMaxCompatibilityLevel()` return null.
  - `prepare()`, `beginPhase()` and `checkEnv()` do nothing.
- `offer(internal)` walks `getInterfaces()` of the concrete class and, recursively, of each interface it finds. Superclasses are not walked. Each type that directly lists `IMixinInternal` among its interfaces becomes a key for `internal` [MixinServiceAbstract.java:registerInternal].
- `getInternal(type)` returns the first registered internal, in `HashMap` order, whose key type is assignable to `type`, or null if there is none [MixinServiceAbstract.java:getInternal].
- Service agents come from `getServiceAgents()`, which runs once, lazily [MixinServiceAbstract.java:getServiceAgents]:
  - It resolves each `getPlatformAgents()` name with `getClassProvider().findClass(name, false)` and calls `getDeclaredConstructor().newInstance()`.
  - It keeps only `IMixinPlatformServiceAgent` instances. Any `Exception` is printed and the agent is skipped.
  - These instances are separate from the per-container agents that `MixinContainer` creates in src/main/java/org/spongepowered/asm/launch/platform.
- `init()`, `wire()` and `unwire()` forward to every service agent [MixinServiceAbstract.java:init] [MixinServiceAbstract.java:wire] [MixinServiceAbstract.java:unwire].
- `getMixinContainers()` returns an `ImmutableList` of every non-null `agent.getMixinContainers()`, in agent order [MixinServiceAbstract.java:getMixinContainers].
- `getSideName()` is final [MixinServiceAbstract.java:getSideName]:
  - It returns the first non-null `agent.getSideName()` and caches it.
  - An agent exception goes to `getLogger("mixin").catching(ex)` and the next agent is asked.
  - If no agent answers, it returns `Constants.SIDE_UNKNOWN` (`"UNKNOWN"`) without caching it.
- `getLogger(name)` is synchronized. It calls `createLogger(name)` once per name and caches the result in a static map shared by every `MixinServiceAbstract` instance and subclass. The default `createLogger` returns `new LoggerAdapterDefault(name)`, which discards all output [MixinServiceAbstract.java:getLogger] [MixinServiceAbstract.java:createLogger].
- `getReEntranceLock()` returns the instance's `new ReEntranceLock(1)` [MixinServiceAbstract.java:lock].

### Throwable usage

- `ServiceInitialisationException` is the unchecked signal a bootstrap service throws when its host is missing [ServiceInitialisationException.java:ServiceInitialisationException].
- `ServiceVersionError` is thrown by `MixinPlatformManager.scanForContainers` in src/main/java/org/spongepowered/asm/launch/platform when the service raises `AbstractMethodError` [ServiceVersionError.java:ServiceVersionError].

## Side Effects

- Static state, none of which can be reset:
  - `MixinService.instance` [MixinService.java:instance]
  - `MixinService.logBuffer`, whose methods are synchronized [MixinService.java:logBuffer]
  - `MixinServiceAbstract.loggers` [MixinServiceAbstract.java:loggers]
- Class loading [MixinService.java:runBootServices] [MixinService.java:initService] [MixinServiceAbstract.java:getServiceAgents]:
  - Every registered `IMixinServiceBootstrap` is created and its `bootstrap()` is called.
  - `IMixinService` providers are created in order until one is valid.
  - Platform service agents are loaded through the service's class provider.
- Console output:
  - When service startup throws an `Error`, `LoggerAdapterConsole` writes to `System.err` [MixinService.java:getDefaultLogger].
  - `getServiceAgents` prints stack traces for agents that fail to load [MixinServiceAbstract.java:getServiceAgents].

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileJava`
- Run from the repo root: `mise exec -- ./gradlew test`
- `build.gradle` declares a `test` source set, but `src/test` does not exist yet. `test` therefore runs no tests today.
