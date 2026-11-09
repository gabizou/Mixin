# SpongePowered Mixin Blueprint

## System Overview

Mixin is a trait/mixin framework for Java built on ASM. It hooks into runtime class loading through pluggable host services (README.MD). Consumers write annotated mixin classes (`@Mixin`, `@Shadow`, `@Overwrite`, `@Inject`, `@Redirect` and others) and list them in JSON mixin configs. At runtime, the launcher's class-transformation hook hands each target class to the Mixin transformer, which merges the mixin bytecode into it. At compile time, the annotation processor in `src/ap` writes refmaps and obfuscation mappings.

This file is the EGM contract index. The human-written project README is `README.MD`, which is left unchanged; on this case-insensitive volume it is the same path as `README.md`.

### Boot flow per launcher

- **LegacyLauncher (`src/launchwrapper`)**:
  1. The `MixinTweaker` constructor calls `MixinBootstrap.start()`.
  2. `MixinService.boot()` runs every `IMixinServiceBootstrap` provider and selects the first valid `IMixinService`, which here is `LaunchWrapper`.
  3. `MixinEnvironment.init(initialPhase)` runs, and `beginPhase()` registers the `Proxy` `IClassTransformer`.
  4. `acceptOptions` calls `MixinBootstrap.doInit`, which prepares the platform. `service.init()` then queues `EnvironmentStateTweaker`.
  5. `injectIntoClassLoader` calls `MixinBootstrap.inject()`.
  6. `EnvironmentStateTweaker.getLaunchArguments()` calls `MixinEnvironment.gotoPhase(DEFAULT)`.

  See `src/launchwrapper/java/org/spongepowered/asm/launch/EGM.md`.
- **ModLauncher (`src/modlauncher*`)**:
  1. The `mixin` `ITransformationService.initialize` requires the `mixin` `ILaunchPluginService`, which must be a `MixinLaunchPluginLegacy`.
  2. It calls `MixinBootstrap.start()` and then `plugin.init`, which calls `service.onInit`.
  3. `initializeLaunch` calls `doInit`, then `inject`, then `onStartup`, which moves the environment to `Phase.DEFAULT`.

  ModLauncher 4–8 registers through `src/modlauncher4/resources/META-INF/services`. ModLauncher 9+ registers through the `provides` clauses in `src/modularity/java/module-info.java`. See `src/modlauncher/java/org/spongepowered/asm/launch/EGM.md`.

### Transformation pipeline

`MixinTransformer` → `MixinProcessor.applyMixins` → config selection and preparation → `TargetClassContext` → `MixinApplicatorStandard` passes `MAIN`, `INJECT_PREPARE`, `ACCESSOR`, `INJECT_PREINJECT`, `INJECT_APPLY` → `postApply` → extensions (export, checks). See `src/main/java/org/spongepowered/asm/mixin/transformer/EGM.md`.

## Subdirectories & System Boundaries

- src: [./src/EGM.md](./src/EGM.md). Twelve Gradle source sets: `main` (the core runtime), `legacy`, `launchwrapper`, `modlauncher`, `modlauncher4`, `modlauncher9`, `modularity`, `ap`, `example`, `agent`, `fernflower` and `bridge`.
- `buildSrc/` (not documented): Groovy build logic for the potemkin module jars (`org.spongepowered.asm.mixin.build`). The EGM bootstrap skips it because its package directory is named `build`.

## Global Invariants

- **Phase order** is `PREINIT(0)` < `INIT(1)` < `DEFAULT(2)`. `MixinEnvironment.init` takes effect only once. `gotoPhase` calls `IMixinService.beginPhase()` only when moving forward (`src/main/java/org/spongepowered/asm/mixin/EGM.md`).
- **Version handshake**: constructing an environment throws `MixinException` unless the global property `mixin.initialised` equals `MixinBootstrap.VERSION` (`"0.8.7"`). `MixinBootstrap.start()` throws `MixinInitialisationError` if a different version already registered (`src/main/java/org/spongepowered/asm/launch/EGM.md`).
- **Service selection**: `ServiceLoader` runs on `MixinService`'s own class loader, and the first `IMixinService` whose `isValid()` returns true wins. If none is valid it throws `ServiceNotAvailableError`. There is no built-in fallback service (`src/main/java/org/spongepowered/asm/service/EGM.md`).
- **Single transformer**: only one `MixinTransformer` may be active; a second instance throws `MixinException("Terminating MixinTransformer instance")`. Re-entrant transformation throws `ReEntrantTransformerError`, and loading a class from a mixin package throws `IllegalClassLoadError` (`.../mixin/transformer/EGM.md`).
- **Ordering**: mixins apply in ascending `priority` order, with a default of 1000, so lower priorities apply first. Injectors apply in ascending `@InjectorOrder` order (EARLY 0, DEFAULT 1000, LATE 2000, REDIRECT 10000, AFTER_REDIRECT 20000) across all mixins on a target (`.../mixin/injection/struct/EGM.md`).
- **Failure semantics**: a failure in a required config or mixin is fatal, raising `MixinApplyError`, `MixinInitialisationError` or `MixinTransformerError`. A failure in a non-required one is logged, and that mixin or config is dropped. `IMixinErrorHandler` can change the action (`.../mixin/extensibility/EGM.md`).
- **Static state is never reset**: `MixinEnvironment` (current phase, compatibility, header latch), `Mixins` (registered names, error handlers), `MixinBootstrap`, `MixinService.instance`, the `ClassInfo` cache, `MixinConfig`/`MixinInfo` order counters, and the `InjectionPoint`, `InjectionInfo` and `TargetSelector` registries. No method clears any of it. **Implication for tests:** each integration scenario needs a fresh class loader (or a forked JVM), plus a test `IMixinService` and `IGlobalPropertyService` registered through `META-INF/services`.

## Global Side Effects

- **Global properties**: everything goes through `GlobalProperties` into the active `IGlobalPropertyService`, which is `Launch.blackboard` (LaunchWrapper) or `Launcher.INSTANCE.blackboard()` (ModLauncher). Keys include `mixin.initialised`, `mixin.platform`, `mixin.agents` and `mixin.configs.*`.
- **System properties**: Mixin reads `mixin.*` options (`MixinEnvironment.Option`). It writes `mixin.hotSwap=true` (agent `premain`), `mixin.env.remapRefMap=true` (LaunchWrapper under GradleStart) and `mixin.target.mapid` (the annotation processor).
- **Filesystem**:
  - `.mixin.out/class` is recursively deleted and rewritten when debug export is on.
  - `.mixin.out/java` receives the Fernflower output.
  - `.mixin.out/audit/mixin_implementation_report.{csv,txt}` is written by the interface checks.
  - `<name>.target.class` is dumped on failure (`mixin.dumpTargetOnFailure`).
  - The annotation processor writes refmap and SRG/TSRG/JSON mapping files and `${java.io.tmpdir}/mixin-targetdb-<id>.tmp`.
- **Console**: `System.err` gets `PrettyPrinter` tables for the startup banner, the verbose, export and profiler options, `LocalCapture.PRINT`, and audits.
- **Class loading and instrumentation**: bytecode is loaded through the host service's `IClassBytecodeProvider`. The hot-swap agent calls `Instrumentation.redefineClasses` on live classes.

## Known Code Quirks (Test Candidates)

These are current behaviours that a test suite should pin down or report upstream. Each is cited in the child contract named beside it.

Checked by the root orchestrator against source:

- `Bytecode.getSimpleName(Type)` passes a dotted class name to `getSimpleName(String)`. That method finds no `/`, so `substring(1)` drops the first character (`.../util/EGM.md`).
- `InjectionPoint.Intersection.find` never sets `in = false`, so it returns every node of the first component (`.../mixin/injection/EGM.md`).
- `ElementNode` (invokedynamic) checks `bsmArgs.length > 1` but reads `bsmArgs[2]` (`.../mixin/injection/selectors/EGM.md`).
- `ContainerHandleVirtual.equals` returns true only for a `String`, so `x.equals(x)` is false (`.../launch/platform/container/EGM.md`).
- `MappingMethod.equals` ignores the owner, but `hashCode` includes it (`.../obfuscation/mapping/common/EGM.md`).
- `MemberRef.Method` and `MemberRef.Field` `setOpcode` reject only opcodes that share no bits with the valid set (`.../mixin/struct/EGM.md`).
- `MixinPlatformAgentFMLLegacy.getPhaseProvider()` names a nested `$PhaseProvider` class that does not exist (`src/launchwrapper/.../launch/platform/EGM.md`).
- `CompatibilityLevel.maxCompatibleLevel` is never assigned, so `canElevateTo` and `canSupport` always return true (`.../mixin/EGM.md`).

Reported by workers but not re-checked here:

- `ClassSignature.merge` turns `Z` into `[`, and `PrettyPrinter.addWrapped` drops a character on hard wraps. Both were observed in jshell (`.../util/EGM.md`).
- `ElementNode.equals` is asymmetric, so `TargetSelector.runSelector` never de-duplicates. `MemberMatcher` uses only its last pattern (`.../mixin/injection/selectors/EGM.md`).

## System Verification

- Run from the repository root with the mise-pinned JDKs (`mise.toml`: Zulu 16 runs Gradle 7.0.2, and Zulu 8 builds the 1.6/1.8 source sets):
  - `mise exec -- ./gradlew compileJava compileLegacyJava compileApJava compileAgentJava compileBridgeJava compileExampleJava compileFernflowerJava compileLaunchwrapperJava compileModlauncherJava compileModlauncher4Java compileModlauncher9Java compileModularityJava`
  - `mise exec -- ./gradlew test`. The `test` source set and `compileTestJava` task exist, but `src/test/` does not, so there are no tests today.
