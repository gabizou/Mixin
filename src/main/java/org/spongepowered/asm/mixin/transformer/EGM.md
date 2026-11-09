# transformer

## Overview

This package is the internal mixin transformation pipeline. `MixinTransformer`
is the single active `IMixinTransformer`. It builds the parts of the pipeline,
then hands class transformation to `MixinProcessor` and class generation to
`MixinClassGenerator`.

`MixinProcessor` does four things:

- selects pending `MixinConfig`s from the global `Mixins` registry for each `MixinEnvironment`
- prepares the selected configs in ordered passes
- matches each target class to its sorted `MixinInfo`s
- applies them through coprocessors, `TargetClassContext`, the applicators (`MixinApplicatorStandard` and `MixinApplicatorInterface`) and the `Extensions` hooks

`ClassInfo` is the process-wide class-metadata cache that all of these steps
query. The package also owns re-entrance detection, illegal mixin-class loads,
error routing by phase, the audit trail and debug export.

## Interface

Entry points:

- `IMixinTransformer` declares these methods [IMixinTransformer.java:IMixinTransformer]:
  - `audit(MixinEnvironment)`
  - `reload(String, ClassNode)`
  - `computeFramesForClass(...)`
  - `transformClassBytes(String name, String transformedName, byte[])`
  - `transformClass(env, name, byte[] | ClassNode)`
  - `generateClass(env, name [, ClassNode])`
  - `getExtensions()`
- `IMixinTransformerFactory.createTransformer() throws MixinInitialisationError`. `MixinTransformer.Factory` implements it [IMixinTransformerFactory.java:createTransformer] [MixinTransformer.java:Factory]
- `MixinTransformer` is package-private and final, and extends `TreeTransformer`. Its constructor builds the parts in this order [MixinTransformer.java:MixinTransformer]:
  1. `SyntheticClassRegistry`
  2. `Extensions`
  3. the hot-swap agent
  4. `MixinCoprocessorNestHost`
  5. `MixinProcessor`
  6. `MixinClassGenerator`
  7. `DefaultExtensions.create(...)`
- `MixinProcessor` (package-private):
  - `synchronized applyMixins(MixinEnvironment, String, ClassNode)`
  - `audit(MixinEnvironment)`
  - `reload(String, ClassNode)`
  - nested enum `ErrorPhase {PREPARE, APPLY}`

  [MixinProcessor.java:applyMixins] [MixinProcessor.java:ErrorPhase]
- `MixinClassGenerator.generateClass(env, name, ClassNode)` is `synchronized` [MixinClassGenerator.java:generateClass]
- `DefaultExtensions.create(env, extensions, registry, nestHost)` registers, in this order:
  1. `ArgsClassGenerator`
  2. `InnerClassGenerator`
  3. `ExtensionClassExporter`
  4. `ExtensionCheckClass`
  5. `ExtensionCheckInterfaces`

  [DefaultExtensions.java:create]

Configuration and mixin model:

- `Config` is the public handle for a `MixinConfig`, identified by name:
  - `static create(String, MixinEnvironment, IMixinConfigSource)` is `@Deprecated` and uses the cache. `create(String, IMixinConfigSource)` does not.
  - Accessors: `getName()`, `getConfig()`, `getEnvironment()`, `getParent()`, `isVisited()`.
  - `equals` and `hashCode` use the name only.

  [Config.java:Config] [Config.java:create]
- `MixinConfig` (package-private, final) is Gson-deserialised. Its JSON keys are:
  - `parent`, `target`, `minVersion`, `requiredFeatures`, `compatibilityLevel`, `required`
  - `priority`, `mixinPriority`, `package`, `mixins`, `client`, `server`
  - `setSourceFile`, `refmap`, `refmapWrapper`, `verbose`, `plugin`
  - `injectors` (`InjectorOptions`): `defaultRequire`, `defaultGroup`, `namespace`, `injectionPoints`, `dynamicSelectors`, `maxShiftBy`
  - `overwrites` (`OverwriteOptions`): `conformVisibility`, `requireAnnotations`

  `MixinConfig.IListener` declares `onPrepare(MixinInfo)` and `onInit(MixinInfo)` [MixinConfig.java:MixinConfig] [MixinConfig.java:InjectorOptions] [MixinConfig.java:OverwriteOptions] [MixinConfig.java:IListener]
- `MixinInfo` is package-private and implements `Comparable<MixinInfo>` and `IMixinInfo`:
  - Lifecycle: constructor, then `parseTargets()`, then `validate()`, then `createContextFor(TargetClassContext)`, `preApply` and `postApply`. `reloadMixin(ClassNode)` handles hot swap.
  - Nested types: `Variant {STANDARD, INTERFACE, ACCESSOR, PROXY}`, `SubType` (`Standard`, `Interface`, `Accessor`), `State` and `Reloaded`, `MixinClassNode`, `MixinMethodNode`, `DeclaredTarget`.

  [MixinInfo.java:MixinInfo] [MixinInfo.java:SubType] [MixinInfo.java:Variant]
- `PluginHandle` is a null-safe wrapper around `IMixinConfigPlugin`. It exposes `onLoad`, `getRefMapperConfig`, `shouldApplyMixin`, `getMixins`, `preApply` and `postApply`, plus legacy reflection through `CompatibilityMode {NORMAL, COMPATIBLE, FAILED}` [PluginHandle.java:PluginHandle] [PluginHandle.java:CompatibilityMode]
- `InterfaceInfo.fromAnnotation(MixinInfo, AnnotationNode)` handles `@Interface` soft-implements: it takes a prefix and renames prefixed methods [InterfaceInfo.java:fromAnnotation] [InterfaceInfo.java:renameMethod]

Class metadata:

- `ClassInfo` (public, final) is a static registry. Its factory and lookup methods are `forName`, `forType`, `forDescriptor`, `fromCache`, `getCommonSuperClass` and `getCommonSuperClassOrInterface`. `fromClassNode` is package-private [ClassInfo.java:forName] [ClassInfo.java:fromClassNode]
- `ClassInfo` hierarchy queries: `hasSuperClass`, `findSuperClass`, `findMethodInHierarchy`, `findFieldInHierarchy`, `findMethod`, `findField`, `getInterfaceMethods`, `hasMixinInHierarchy`, `hasMixinTargetInHierarchy` [ClassInfo.java:findInHierarchy]
- Flag constants: `INCLUDE_PRIVATE`, `INCLUDE_STATIC`, `INCLUDE_ALL`, `INCLUDE_INITIALISERS` (`0x40000`). Enums: `SearchType {ALL_CLASSES, SUPER_CLASSES_ONLY}`, `TypeLookup {DECLARED_TYPE, ELEMENT_TYPE}`, `Traversal {NONE, ALL, IMMEDIATE, SUPER}` [ClassInfo.java:INCLUDE_ALL] [ClassInfo.java:Traversal]
- Member model: `ClassInfo.Member`, with subclasses `Method`, `InterfaceMethod` and `Field`. It tracks the original and current name and descriptor, and the unique, decorated-final, decorated-mutable and conformed flags [ClassInfo.java:Member] [ClassInfo.java:Method.conform]

Application:

- `TargetClassContext` holds the state for one target and one pass:
  - `applyMixins()` runs the application.
  - `getSuppressedExceptions()` returns the failures that were set aside.
  - `isExported()` and `isExportForced()` report whether to export.
  - `getTargetMethod(MethodNode)` returns the target method.
  - `processDebugTasks()` runs the debug tasks.

  [TargetClassContext.java:TargetClassContext]
- `MixinTargetContext` (public) handles one mixin applied to one target:
  - Rewriting: `transformMethod`, `transformDescriptor`
  - Unique names: `getUniqueName`
  - Injectors: `prepareInjections`, `applyPreInjections`, `applyInjections(int)`
  - Accessors: `generateAccessors`
  - Merging: `methodMerged`

  [MixinTargetContext.java:MixinTargetContext]
- `MixinApplicatorStandard` and its subclass `MixinApplicatorInterface`: `apply(SortedSet<MixinInfo>)`, plus the enum `ApplicatorPass {MAIN, INJECT_PREPARE, ACCESSOR, INJECT_PREINJECT, INJECT_APPLY}` [MixinApplicatorStandard.java:ApplicatorPass] [MixinApplicatorInterface.java:MixinApplicatorInterface]
- `MixinPreProcessorStandard` has the subclasses `MixinPreProcessorInterface` and `MixinPreProcessorAccessor`, the latter an empty subclass of the interface variant. Its passes are `prepare(Extensions)`, `conform(...)`, `createContextFor(TargetClassContext)` and `attach(MixinTargetContext)`. `SpecialMethod` enum: `MERGE`, `OVERWRITE`, `SHADOW`, `ACCESSOR`, `INVOKER` [MixinPreProcessorStandard.java:MixinPreProcessorStandard]
- `MixinCoprocessor` and the list `MixinCoprocessors`. The enum `ProcessResult {NONE, TRANSFORMED, PASSTHROUGH_NONE, PASSTHROUGH_TRANSFORMED}` is combined by logical OR in `with(...)`. Concrete coprocessors: `MixinCoprocessorPassthrough`, `MixinCoprocessorSyntheticInner`, `MixinCoprocessorAccessor` and `MixinCoprocessorNestHost` [MixinCoprocessor.java:ProcessResult] [MixinCoprocessors.java:process]

Support:

- `ActivityStack implements IActivityContext`, the activity trail used in exception messages. Glue `" -> "` [ActivityStack.java:ActivityStack]
- `MethodMapper` generates handler and unique names [MethodMapper.java:getHandlerName] [MethodMapper.java:getUniqueName]
- `ClassContext` upgrades `INVOKESPECIAL` to `INVOKEVIRTUAL` for methods whose visibility was raised [ClassContext.java:upgradeMethodRef]
- `InnerClassGenerator` is the `IClassGenerator` named `"inner"`. It makes one renamed copy of each mixin inner class per target [InnerClassGenerator.java:registerInnerClass]
- `SyntheticClassRegistry` and `SyntheticClassInfo` are the registry of generated classes, keyed by internal name [SyntheticClassRegistry.java:registerSyntheticClass] [SyntheticClassInfo.java:SyntheticClassInfo]

Child packages:

- [ext](./ext/EGM.md): the extension SPI and the `Extensions` registry. Child [ext/extensions](./ext/extensions/EGM.md) holds the class checker, the interface audit and the class exporter.
- [meta](./meta/EGM.md): the bytecode markers `@MixinMerged`, `@MixinRenamed`, `@MixinProxy` and `@MixinInner`.
- [struct](./struct/EGM.md): `InsnRange` and `Initialiser`, which handle constructor field initialisers.
- [throwables](./throwables/EGM.md): `InvalidMixinException` and its subtypes, and `MixinTransformerError` and its subtypes.

## Invariants

### Config lifecycle and phases

- `MixinConfig.create` throws `IllegalArgumentException("The specified resource '<f>' was invalid or could not be read")` in two cases: the resource is missing, or loading throws an `Exception` (which it wraps). A `MixinInitialisationError` passes through unwrapped. `create` returns `null` when `onLoad` fails a version or feature check on a non-required config [MixinConfig.java:create]
- `onLoad` defers to `assignParent` when `parent` is set. Otherwise it resolves the environment from `target` and sets `required` to `required && !Option.IGNORE_REQUIRED`. It also replaces a priority below 0 with `1000`, then calls `postInit` [MixinConfig.java:onLoad] [MixinConfig.java:initPriority]
- `postInit` throws `MixinInitialisationError` when:
  - it runs twice (`"...was already initialised."`)
  - `compatibilityLevel` is unknown, too old, or prohibited
  - a required config's `minVersion` is newer than the environment version
  - a required config lists missing `requiredFeatures`

  A non-required config that fails the version or feature check only logs a WARN and is dropped [MixinConfig.java:postInit] [MixinConfig.java:initCompatibilityLevel] [MixinConfig.java:checkVersion] [MixinConfig.java:checkFeatures]
- `assignParent` throws `MixinInitialisationError` in three cases: the parent is already set, the config is its own parent, or the parent is uninitialised (an indirect loop) [MixinConfig.java:assignParent]
- `MixinProcessor.checkSelect` re-selects configs when either condition holds [MixinProcessor.java:checkSelect]:
  - the environment changed
  - there are unvisited configs and nothing has been transformed yet in this phase
- `select` order: `selectConfigs`, then `extensions.select(env)`, then `prepareConfigs`, then `transformedCount = 0` [MixinProcessor.java:select]
- `selectConfigs` removes each selected config from the global `Mixins` iterator and calls `onSelect()`, which creates the `PluginHandle` and resolves the refmap. It then sorts `pendingConfigs` [MixinProcessor.java:selectConfigs] [MixinConfig.java:onSelect]
- `prepareConfigs` makes four passes over the sorted pending configs [MixinProcessor.java:prepareConfigs]:
  1. Registers the coprocessors (and the hot-swap listener) as `IListener`s.
  2. Calls `config.prepare(extensions)`.
  3. Calls `plugin.acceptTargets(own, others)`.
  4. Calls `config.postInitialise(extensions)`.
- `MixinConfig.prepare` runs once. It always loads `mixins`, plus `client` or `server` according to `env.getSide()`. An unknown side logs a WARN and loads no sided mixins [MixinConfig.java:prepare]
- `postInitialise` loads the companion plugin's `getMixins()` and calls `validate()` and `onInit` on every mixin. A failed mixin is logged at ERROR and removed. It is never rethrown, even for required configs [MixinConfig.java:postInitialise]
- `MixinInfo.validate()` throws `IllegalStateException` when there is no pending state. `reloadMixin` throws `IllegalStateException("Cannot reload mixin while it is initialising")` [MixinInfo.java:validate] [MixinInfo.java:reloadMixin]
- A reload that changes any of these throws `MixinReloadException`: inner classes, interfaces, soft interfaces, targets, or priority [MixinInfo.java:Reloaded.validateChanges]

### Mixin target selection

- A mixin's class name is `config package + name`. A missing class becomes `InvalidMixinException("The specified mixin '<name>' was not found")` [MixinInfo.java:loadMixinClass]
- A missing `@Mixin` throws `InvalidMixinException("The mixin '<cls>' is missing an @Mixin annotation")` [MixinInfo.java:readPriority]
- Targets are read from `@Mixin.value`, which holds public `Type` targets, followed by `@Mixin.targets`, which holds private `String` targets remapped through the refmap [MixinInfo.java:readTargets] [MixinInfo.java:DeclaredTarget.of]
- A declared target is kept only if `plugin.shouldApplyMixin(target, mixin)` returns true. With no plugin, the answer is `true` [MixinInfo.java:shouldApplyMixin] [PluginHandle.java:shouldApplyMixin]
- If the `IClassTracker` reports a target as already loaded and the mixin is not reloading, the mixin logs `"Critical problem: ... was loaded too early."`. A required config throws `MixinTargetAlreadyLoadedException` instead [MixinInfo.java:readDeclaredTargets]
- When `ClassInfo.forName` cannot resolve a target, the target is excluded. A `@Pseudo` mixin logs at DEBUG; any other mixin goes to `handleTargetError`. `handleTargetError` throws `InvalidMixinException` only when `Option.DEBUG_TARGETS` is set, and otherwise logs a WARN [MixinInfo.java:getTargetClass] [MixinInfo.java:handleTargetError]
- The target kind must match the subtype:
  - An interface target on a standard mixin, or a class target on an interface mixin, throws `InvalidMixinException("@Mixin target type mismatch ...")`.
  - The subtype comes from `getVariant`: a non-interface is `STANDARD`; an interface with any method that is neither accessor nor synthetic is `INTERFACE`; anything else is `ACCESSOR`. `PROXY` is never returned.

  [MixinInfo.java:SubType.validateTarget] [MixinInfo.java:getVariant]
- A mixin that resolves zero targets is silently not added. Mixin names that are already in the static `globalMixinList` are skipped [MixinConfig.java:prepareMixins]

### Application order and priority

- `MixinInfo.compareTo` and `MixinConfig.compareTo` sort by ascending `priority`. Ties go by construction order, from the static counters `mixinOrder` and `configOrder` [MixinInfo.java:compareTo] [MixinConfig.java:compareTo]
- `applyMixins` merges every config's mixins for the target into a `TreeSet<MixinInfo>`, so lower-priority mixins are applied first [MixinProcessor.java:applyMixins]
- The `@Mixin.priority` default is the config's `mixinPriority`, and that default is `1000` [MixinInfo.java:readPriority] [MixinConfig.java:initPriority]
- An incoming method skips a target method that already carries `@MixinMerged` with `priority >= incoming` from a different mixin, and logs a WARN `"Method overwrite conflict"`. So a later mixin overwrites only when its priority is strictly higher [MixinApplicatorStandard.java:isAlreadyMerged]
- A `@MixinMerged` whose `sessionId` differs from the current session throws `ClassFormatError("Invalid @MixinMerged annotation found in...")`. A `@Final` target is never overwritten; this logs a WARN [MixinApplicatorStandard.java:isAlreadyMerged]
- `MixinApplicatorStandard.apply` runs in this order [MixinApplicatorStandard.java:apply]:
  1. `createContextFor` for each mixin. This runs the preprocessor's `prepare`, `conform` and `attach`.
  2. `preApply` for each mixin. This calls the plugin's `preApply`.
  3. Each `ApplicatorPass` in declaration order. Within a pass, each injector order runs across all mixins before the next order starts. `INJECT_APPLY` uses an ascending `TreeSet` of injector orders; every other pass uses `{0}`.
  4. `postApply`.
  5. `applySourceMap`.
  6. `processDebugTasks`.
- The `MAIN` pass runs these steps in order [MixinApplicatorStandard.java:applyMixin]:
  1. signature
  2. interfaces
  3. attributes
  4. annotations
  5. fields (shadow fields before new fields)
  6. methods (shadow methods before normal methods)
  7. initialisers
- `TargetClassContext.applyMixins` throws `IllegalStateException("Mixins already applied to target class ...")` on a second call. It runs `extensions.preApply`, then the applicator, then `applySignature`, `upgradeMethods` and `checkMerges`, then `extensions.postApply` [TargetClassContext.java:applyMixins] [TargetClassContext.java:apply]
- The applicator is `MixinApplicatorInterface` when the target is an interface, and `MixinApplicatorStandard` otherwise [TargetClassContext.java:createApplicator]
- Coprocessors run in this order: `Passthrough`, `SyntheticInner`, `Accessor`, `NestHost`. `process` and `postProcess` never short-circuit [MixinProcessor.java:MixinProcessor] [MixinCoprocessors.java:process]
- In a passthrough result, every coprocessor's `postProcess` runs, then `export`, and no config lookup happens [MixinProcessor.java:applyMixins]

### Validation failures and error routing

- A failure from a non-required mixin during the applicator passes goes to `addSuppressed`, and the mixin is dropped from later passes. A required mixin's failure is rethrown. Any non-`InvalidMixinException` is wrapped as `MixinApplicatorException("Unexpecteded ... whilst applying the mixin class:")`; the typo is in the source [MixinApplicatorStandard.java:apply]
- `MixinProcessor.handleMixinError` decides what to do with an error [MixinProcessor.java:handleMixinError]:
  - It defaults to `ERROR` if the config is required and `WARN` otherwise.
  - Each `IMixinErrorHandler` may override that action.
  - `ERROR` throws `MixinApplyError` in both the `PREPARE` and `APPLY` phases.
  - An `InvalidMixinException` with a `null` mixin is rethrown and leaves `errorState = true`. While `errorState` is true, `applyMixins` returns `false`.
- Any non-`MixinTransformerError` `Throwable` during application becomes `MixinTransformerError("An unexpected critical error was encountered")`, and that includes a `MixinApplyError` from a required failure [MixinProcessor.java:applyMixins]
- `applyMixins` throws `ReEntrantTransformerError("Re-entrance error.")` in two cases. On the prepare-phase path it throws before the `try`, so `lock.pop()` is skipped. On the apply path the `finally` pops the lock [MixinProcessor.java:applyMixins]
- Loading a class inside a config's mixin package throws `IllegalClassLoadError`, unless the class is an `InjectionPoint` or an `ITargetSelectorDynamic` subclass [MixinProcessor.java:applyMixins] [MixinProcessor.java:getInvalidClassError]
- Only one transformer may exist. If an `IMixinTransformer` is already active, the constructor throws `MixinException("Terminating MixinTransformer instance ...")` [MixinTransformer.java:MixinTransformer]
- Preprocessor rejections all throw `InvalidMixinException` [MixinPreProcessorStandard.java:attachSpecialMethod] [MixinPreProcessorStandard.java:attachFields] [MixinPreProcessorStandard.java:validateField] [MixinPreProcessorStandard.java:conformVisibility]:
  - `@Shadow` targets that are not found, and aliases of `<init>`
  - mismatched `STATIC` modifiers, and conflicting field descriptors
  - aliasing a non-private member
  - non-private static fields, and shadow-prefixed fields
  - an overwrite that reduces visibility without `conformVisibility`
  - a `@Unique` conflict when `Option.DEBUG_UNIQUE` is set
- Accessor and invoker misuse throws `InvalidAccessorException` [MixinPreProcessorStandard.java:attachAccessorMethod]
- Other failures in `prepare`, `conform` or `attach` are wrapped in `MixinPreProcessorException`. `attach` runs only once and throws `IllegalStateException` on a second call [MixinPreProcessorStandard.java:attach]
- `mergeMethod` throws `InvalidMixinException` when an `@Overwrite` target is missing, or when the config requires `@Overwrite` and the method lacks it [MixinApplicatorStandard.java:mergeMethod]
- `checkMethodVisibility` rejects non-private static methods [MixinApplicatorStandard.java:checkMethodVisibility]
- `checkConstraints` rejects constraint violations, unless `Option.IGNORE_CONSTRAINTS` is set [MixinApplicatorStandard.java:checkConstraints]
- `MixinApplicatorInterface` and `MixinPreProcessorInterface` throw `InvalidInterfaceMixinException` for the following [MixinApplicatorInterface.java:prepareInjections] [MixinPreProcessorInterface.java:prepareMethod] [MixinPreProcessorInterface.java:validateField]:
  - injectors, while `Feature.INJECTORS_IN_INTERFACE_MIXINS` is disabled
  - private methods below the required compatibility level
  - instance fields
- `MixinInfo` validation throws `InvalidMixinException` in these cases [MixinInfo.java:State.validateInner] [MixinInfo.java:State.validateClassFeatures] [MixinInfo.java:State.validateRemappables] [MixinInfo.java:SubType.Standard.validate]:
  - a non-static inner mixin
  - unsupported language features
  - a remappable `@Shadow` in a multi-target mixin
  - a superclass that is not in the target hierarchy
- `MixinTargetContext` throws `InvalidMixinException` for these [MixinTargetContext.java:requireVersion] [MixinTargetContext.java:validateMethod] [MixinTargetContext.java:checkFinal]:
  - an unsupported class version
  - an override of a final method
  - `@SoftOverride` misuse
  - a write to a `@Final` shadow field, but only when `Option.DEBUG_VERIFY` is set

### Static and global registries

- `ClassInfo.cache` is a static, unsynchronised `HashMap` keyed by internal name and seeded only with `java/lang/Object` [ClassInfo.java:cache] [ClassInfo.java:OBJECT]
- `forName` caches `null` on a failed load. The `== null` check cannot tell a cached failure from a missing entry, so it retries and logs again on every call [ClassInfo.java:forName]
- `fromClassNode` returns the existing entry and ignores the new `ClassNode` [ClassInfo.java:fromClassNode]
- `Config.allConfigs` is static. Lookups use the file argument, but entries are stored under the config name [Config.java:allConfigs]
- `MixinConfig.globalMixinList` and `configOrder`, and `MixinInfo.mixinOrder`, are static and never reset [MixinConfig.java:globalMixinList] [MixinInfo.java:mixinOrder]
- `MethodMapper.classes` and `MethodMapper.methods` are static and never cleared, so generated handler names depend on JVM history. Unique names have the form `md<sessionId[30:]>$...` and `fd...` and use per-instance counters [MethodMapper.java:getClassUID] [MethodMapper.java:getUniqueName]
- `SyntheticClassRegistry.registerSyntheticClass` throws `MixinError("Synthetic class with name ... was already registered by ...")` when a different instance uses the same name [SyntheticClassRegistry.java:registerSyntheticClass]
- `InnerClassGenerator` names copies `<target>$<simple>$<random uuid>`. Registering the same owner, inner class and target again is a no-op [InnerClassGenerator.java:getUniqueReference] [InnerClassGenerator.java:registerInnerClass]
- `MixinProcessor.sessionId` is a random UUID for each processor instance [MixinProcessor.java:sessionId]
- `PluginHandle` loads the plugin with `findClass(name, true)`. A failure is logged at ERROR and leaves the plugin `null`. In `FAILED` mode, `preApply` and `postApply` throw `IllegalStateException` [PluginHandle.java:PluginHandle] [PluginHandle.java:preApply]

## Side Effects

- Class loading:
  - `ClassInfo.forName` loads bytecode through `MixinService.getService().getBytecodeProvider().getClassNode(name, true, flags)` [ClassInfo.java:forName]
  - `MixinInfo` loads mixin bytecode the same way [MixinInfo.java:loadMixinClass]
  - `audit` force-loads every unhandled target with `findClass(target, true)` [MixinProcessor.java:audit]
  - `MixinTransformer.initHotSwapper` loads `org.spongepowered.tools.agent.MixinAgent` reflectively when `Option.HOT_SWAP` is set [MixinTransformer.java:initHotSwapper]
  - Error handlers are re-instantiated on every error [MixinProcessor.java:getErrorHandlers]
- Global state:
  - The `MixinTransformer` constructor calls `MixinEnvironment.setActiveTransformer(this)` before it builds anything else [MixinTransformer.java:MixinTransformer]
  - `selectConfigs` removes configs from the global `Mixins` registry [MixinProcessor.java:selectConfigs]
  - `MixinConfig` may call `MixinEnvironment.setCompatibilityLevel` [MixinConfig.java:initCompatibilityLevel]
  - `MixinConfig` registers into the global `InjectionPoint` and `TargetSelector` registries [MixinConfig.java:initInjectionPoint] [MixinConfig.java:initDynamicSelector]
- Audit trail: `IMixinAuditTrail` is notified with `onApply` for each applied mixin, `onGenerate` for each generated class, and `onPostProcess` for passthrough classes [MixinApplicatorStandard.java:apply] [MixinClassGenerator.java:generateClass] [MixinProcessor.java:applyMixins]
- Debug export and file writes:
  - After a successful apply, `extensions.export` runs with `isExportForced()`. `@Debug(export=true)` forces the export [MixinProcessor.java:applyMixins] [TargetClassContext.java:processDebugTasks]
  - On failure, `Option.DUMP_TARGET_ON_FAILURE` dumps `<name>.target.class` through `ExtensionClassExporter.dumpClass` [MixinProcessor.java:dumpClassOnFailure]
  - With `Option.DEBUG_VERBOSE` set, `@Debug(print=true)` textifies bytecode to `System.err` [TargetClassContext.java:processDebugTasks]
- Bytecode mutation:
  - The target `ClassNode` is mutated in place: interfaces, fields, methods, signature, `sourceFile`, `sourceDebug` and version [MixinApplicatorStandard.java:applyMixin] [MixinApplicatorStandard.java:applySourceMap]
  - The mixin `ClassNode` and the shared `ClassInfo.Member` state are mutated by renames and conforms [MixinPreProcessorStandard.java:attachMethods] [ClassInfo.java:Method.conform]
  - `@MixinMerged` and `@MixinRenamed` are added to merged and renamed methods [MixinTargetContext.java:methodMerged] [InterfaceInfo.java:renameMethod]
- Logging: all log output goes to the `"mixin"` logger, except that `audit` uses `"mixin.audit"`. The `Profiler` sections used are `"mixin"`, `"meta"` and `"export"`, and `select` toggles `Profiler.setActive` [MixinProcessor.java:select] [ClassInfo.java:profiler]
- Resource leak: `MixinConfig.create` does not close the config resource stream [MixinConfig.java:create]

## Verification

Run from the repository root:

- `mise exec -- ./gradlew compileJava`
- `mise exec -- ./gradlew test`

The `test` source set is declared in `build.gradle`, but `src/test` does not exist yet, so there are no tests today.
