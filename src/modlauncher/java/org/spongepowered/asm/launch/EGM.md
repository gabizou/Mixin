# modlauncher: org.spongepowered.asm.launch

## Overview

This package holds the ModLauncher entry points that every ModLauncher version shares. `MixinTransformationServiceAbstract` is the base `ITransformationService`. `MixinLaunchPluginLegacy` is the `ILaunchPluginService`, named `"mixin"`, that hands each class to Mixin's `IClassProcessor`s. Version-specific subclasses and their registrations live in the other source sets:

- ModLauncher 4-8: `MixinTransformationServiceLegacy` in `src/modlauncher4`, registered by `src/modlauncher4/resources/META-INF/services/`.
- ModLauncher 9+: `MixinTransformationService` and `MixinLaunchPlugin` in `src/modlauncher9`, registered by `src/modularity/java/module-info.java`.

The `platform` child package holds the Forge side-detection agent and the root container.

## Interface

- `public interface IClassProcessor` (a Mixin interface, not a ModLauncher one) [IClassProcessor.java:IClassProcessor]
  - `EnumSet<Phase> handlesClass(Type classType, boolean isEmpty, String reason)`. Returning `null` is allowed.
  - `boolean processClass(Phase phase, ClassNode classNode, Type classType, String reason)`.
  - `boolean generatesClass(Type classType)`.
  - `boolean generateClass(Type classType, ClassNode classNode)`.
- `public final class Phases`, with constants `NONE` (empty), `BEFORE_ONLY` (`{BEFORE}`) and `AFTER_ONLY` (`{AFTER}`) [Phases.java:Phases].
- `public class MixinLaunchPluginLegacy implements ILaunchPluginService, IClassBytecodeProvider` [MixinLaunchPluginLegacy.java:MixinLaunchPluginLegacy]
  - `public static final String NAME = "mixin"`. `name()` returns it [MixinLaunchPluginLegacy.java:NAME].
  - `handlesClass(Type, boolean, String)` and `processClass(Phase, ClassNode, Type, String)` [MixinLaunchPluginLegacy.java:handlesClass(Type,boolean,String)], [MixinLaunchPluginLegacy.java:processClass(Phase,ClassNode,Type,String)].
  - The overloads without a reason argument, `handlesClass(Type, boolean)` and `processClass(Phase, ClassNode, Type)`, always throw `IllegalStateException("Outdated ModLauncher")` [MixinLaunchPluginLegacy.java:handlesClass(Type,boolean)].
  - `void init(IEnvironment, List<String> commandLineMixins)` is package-private [MixinLaunchPluginLegacy.java:init].
  - `void initializeLaunch(ITransformerLoader, Path[])` covers the ModLauncher 4-8 signature, and `protected void initializeLaunch(ITransformerLoader)` holds the shared logic [MixinLaunchPluginLegacy.java:initializeLaunch(ITransformerLoader,Path[])], [MixinLaunchPluginLegacy.java:initializeLaunch(ITransformerLoader)].
  - Resource intake methods [MixinLaunchPluginLegacy.java:offerResource], [MixinLaunchPluginLegacy.java:addResources]:
    - `@Deprecated addResource(Path, String)`, the ModLauncher 4.0 method.
    - `offerResource(Path, String)`, for ModLauncher 7.0+.
    - `addResources(List)`.
  - `customAuditConsumer(String, Consumer<String[]>)` and `getExtension()`, which returns `null` [MixinLaunchPluginLegacy.java:customAuditConsumer].
  - `getClassNode(String)`, `getClassNode(String, boolean)` and `getClassNode(String, boolean, int)` [MixinLaunchPluginLegacy.java:getClassNode(String,boolean,int)].
- `public abstract class MixinTransformationServiceAbstract implements ITransformationService` [MixinTransformationServiceAbstract.java:MixinTransformationServiceAbstract]
  - `name()` returns `"mixin"` [MixinTransformationServiceAbstract.java:name].
  - `arguments(...)` registers the option `config`, described as "a mixin config to load", which takes a required `String` argument [MixinTransformationServiceAbstract.java:arguments].
  - `argumentValues(OptionResult)` collects the values of that option [MixinTransformationServiceAbstract.java:argumentValues].
  - `onLoad(...)` is empty, so it performs no version check and never throws `IncompatibleEnvironmentException` [MixinTransformationServiceAbstract.java:onLoad].
  - `initialize(IEnvironment)` [MixinTransformationServiceAbstract.java:initialize].
  - `runScan(IEnvironment)` returns an empty list. It exists so that ModLauncher 8 does not call the removed `beginScanning` [MixinTransformationServiceAbstract.java:runScan].
  - `transformers()` returns an empty list, so Mixin registers no ModLauncher `ITransformer` [MixinTransformationServiceAbstract.java:transformers].
- Child package: [platform](./platform/EGM.md) holds `MixinPlatformAgentMinecraftForge`, and its `container` child holds `ContainerHandleModLauncher`.

## Invariants

- **Boot order under ModLauncher.** ModLauncher fixes the callback order. Core steps are cited by path.
  1. **`initialize(env)`** [MixinTransformationServiceAbstract.java:initialize]:
     - It calls `env.findLaunchPlugin("mixin")`. If that is empty, it throws `MixinInitialisationError("Mixin Launch Plugin Service could not be located")`. If the plugin is not a `MixinLaunchPluginLegacy`, it throws `MixinInitialisationError("Mixin Launch Plugin Service is present but not compatible")`.
     - It then calls `MixinBootstrap.start()`. Initializing that class boots the services and `prepare()`s the selected service. `start()` itself calls `MixinEnvironment.init(PREINIT)`, which wires the phase consumer, and then `beginPhase()`.
     - Finally it calls `plugin.init(env, commandLineMixins)`.
  2. **`plugin.init`** requires `MixinService.getService()` to be a `MixinServiceModLauncher`. Otherwise it throws `IllegalStateException("Unsupported service type for ModLauncher Mixin Service")`. It then reads the audit trail, adds `service.getProcessors()` to its processor list, stores the command-line mixins, and calls `service.onInit(this)` [MixinLaunchPluginLegacy.java:init].
  3. **Resource intake.** ModLauncher's resource callbacks go to `service.getPrimaryContainer().addResource(name, path)` or `.addResources(list)`. They dereference the service that `init` stored [MixinLaunchPluginLegacy.java:offerResource], [MixinLaunchPluginLegacy.java:addResources].
  4. **`initializeLaunch`** stores the transformer loader. It then calls `MixinBootstrap.doInit(CommandLineOptions.of(commandLineMixins))`, `MixinBootstrap.inject()`, and `service.onStartup()`. `onStartup()` moves Mixin to `Phase.DEFAULT` [MixinLaunchPluginLegacy.java:initializeLaunch(ITransformerLoader)].
- **`handlesClass` dispatch.** Reason `"mixin"` returns `Phases.NONE` without consulting any processor. For any other reason it returns the union of every non-null processor vote, in list order, under `synchronized(processors)` [MixinLaunchPluginLegacy.java:handlesClass(Type,boolean,String)].
- **`processClass` dispatch.** It calls every processor in list order, without skipping and without filtering by vote, and returns the OR of their results [MixinLaunchPluginLegacy.java:processClass(Phase,ClassNode,Type,String)].
- **Processor order.** It is the order of `MixinServiceModLauncher.getProcessors()`: the transformation handler, then the class tracker [MixinLaunchPluginLegacy.java:processors].
- **`getClassNode` with `runTransformers == false`.** It throws `IllegalArgumentException("ModLauncher service does not currently support retrieval of untransformed bytecode")` [MixinLaunchPluginLegacy.java:getClassNode(String,boolean,int)].
- **`getClassNode` lookup.** Otherwise it looks up bytes in this order [MixinLaunchPluginLegacy.java:getClassNode(String,boolean,int)]:
  1. `transformerLoader.buildTransformedClassNodeFor(name)`.
  2. On `ClassNotFoundException`, the thread context class loader resource `<internal name>.class`.
  3. If the bytes are null or empty, the first processor whose `generatesClass` and `generateClass` both return true.
  4. Otherwise it throws `ClassNotFoundException`.
- **`customAuditConsumer`.** It does nothing when there is no audit trail [MixinLaunchPluginLegacy.java:customAuditConsumer].

## Side Effects

- `arguments` adds the `config` option to ModLauncher's command-line parser [MixinTransformationServiceAbstract.java:arguments].
- `initialize` starts the Mixin subsystem and initializes the service [MixinTransformationServiceAbstract.java:initialize].
- `initializeLaunch` runs Mixin `doInit`, `inject` and the move to the DEFAULT phase [MixinLaunchPluginLegacy.java:initializeLaunch(ITransformerLoader)].
- The resource callbacks mutate the service's root container [MixinLaunchPluginLegacy.java:offerResource].
- `getClassNode` reads class bytes through ModLauncher's `ITransformerLoader` or the thread context class loader [MixinLaunchPluginLegacy.java:getClassNode(String,boolean,int)].

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileModlauncherJava`. This uses a Java 8 toolchain with source and target `1.8` (build.gradle lines 129-133) against `cpw.mods:modlauncher:7.0.0` and `jopt-simple:5.0.4` (build.gradle lines 187-195).
