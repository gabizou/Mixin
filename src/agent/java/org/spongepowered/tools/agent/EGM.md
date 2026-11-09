# org.spongepowered.tools.agent (agent source set)

## Overview

This is the hot-swap agent, built on `java.lang.instrument`. When a debugger
redefines a mixin class, the agent reloads that mixin in the Mixin processor. It
then redefines each target class, starting from that class's original
pre-mixin bytecode and transforming it again with the new mixin.

- `MixinAgent` implements `IHotSwap`
  (src/main/java/org/spongepowered/asm/mixin/transformer/ext/IHotSwap.java).
- The package-private `MixinAgentClassLoader` holds the state that hot-swap
  needs: stub classes that stand in for mixins, and the original bytecode of
  each target class.

The agent is connected to the rest of Mixin in three places outside this
partition:

- The jar manifest names `org.spongepowered.tools.agent.MixinAgent` as both
  `Premain-Class` and `Agent-Class`, with `Can-Redefine-Classes: true` and
  `Can-Retransform-Classes: true` (build.gradle:363-367, task `stagingJar`).
- `initHotSwapper` in
  src/main/java/org/spongepowered/asm/mixin/transformer/MixinTransformer.java
  creates the agent by reflection (constant `MIXIN_AGENT_CLASS`, constructor
  `(IMixinTransformer)`):
  - It does this only when `Option.HOT_SWAP` (property `mixin.hotSwap`) is on.
  - If any `Throwable` is thrown, it logs at INFO "Hot-swap agent could not be
    loaded, hot swapping of mixins won't work. {}: {}" and runs with no
    hot-swapper.
- src/main/java/org/spongepowered/asm/mixin/transformer/MixinProcessor.java
  sends the agent the classes to track:
  - It calls `registerTargetClass(name, targetClassNode)` before applying
    mixins to a target (line 358).
  - It calls `registerMixinClass(mixin.getClassName())` from a config
    `onPrepare` listener (line 527).

## Interface

### MixinAgent

- `public class MixinAgent implements IHotSwap` [MixinAgent.java:MixinAgent]
- `public MixinAgent(IMixinTransformer classTransformer)`
  [MixinAgent.java:MixinAgent(IMixinTransformer)]
- `public static void premain(String arg, Instrumentation instrumentation)` is
  the JVM `-javaagent` entry point [MixinAgent.java:premain].
- `public static void agentmain(String arg, Instrumentation instrumentation)` is
  the entry point when the agent is attached to a running JVM
  [MixinAgent.java:agentmain].
- `public static void init(Instrumentation instrumentation)` [MixinAgent.java:init]
- `public void registerMixinClass(String name)` implements `IHotSwap`. It
  delegates to `MixinAgentClassLoader.addMixinClass`
  [MixinAgent.java:registerMixinClass].
- `public void registerTargetClass(String name, ClassNode classNode)` implements
  `IHotSwap`. It delegates to `MixinAgentClassLoader.addTargetClass`
  [MixinAgent.java:registerTargetClass].
- `public static final byte[] ERROR_BYTECODE = {1}`. Its Javadoc says that
  returning it from the transformer "causes a class file format exception and
  indicates in the ide that somethings went wrong" [MixinAgent.java:ERROR_BYTECODE].
- `public static void log(Level level, String message, Object... params)`
  [MixinAgent.java:log]
- The package-private inner class `Transformer implements ClassFileTransformer`
  has the methods `transform`, `reloadMixin` and `reApplyMixins`
  [MixinAgent.java:Transformer].
- Every agent instance shares two package-private static fields:
  - `classLoader`, a single `MixinAgentClassLoader` [MixinAgent.java:classLoader]
  - `instrumentation` [MixinAgent.java:instrumentation]
- `private static List<MixinAgent> agents` holds every agent constructed
  [MixinAgent.java:agents].

### MixinAgentClassLoader (package-private)

- `class MixinAgentClassLoader extends ClassLoader`
  [MixinAgentClassLoader.java:MixinAgentClassLoader]
- `void addMixinClass(String name)` [MixinAgentClassLoader.java:addMixinClass]
- `void addTargetClass(String name, ClassNode classNode)`
  [MixinAgentClassLoader.java:addTargetClass]
- `byte[] getFakeMixinBytecode(Class<?> clazz)`
  [MixinAgentClassLoader.java:getFakeMixinBytecode]
- `byte[] getOriginalTargetBytecode(String name)`
  [MixinAgentClassLoader.java:getOriginalTargetBytecode]
- `public static void log(Level, String, Object...)` behaves the same as
  `MixinAgent.log` [MixinAgentClassLoader.java:log].

### Hot-swap flow

1. **Startup.**
   - `premain` sets the system property `mixin.hotSwap=true` and calls `init`.
     `agentmain` calls `init` without setting the property. Both ignore `arg`
     [MixinAgent.java:premain] [MixinAgent.java:agentmain].
   - `init` stores the `Instrumentation`. If `isRedefineClassesSupported()` is
     false, it logs at ERROR "The instrumentation doesn't support re-definition
     of classes" and continues anyway. It then registers a `Transformer` for
     every agent constructed so far [MixinAgent.java:init].
   - An agent constructed after `init` registers its own `Transformer` in its
     constructor [MixinAgent.java:MixinAgent(IMixinTransformer)].
   - Each `Transformer` is registered with `addTransformer(transformer, true)`,
     so it can retransform [MixinAgent.java:initTransformer].
2. **Registration.**
   - `addMixinClass(name)` creates a stub class with the mixin's name, defines
     it in this loader and instantiates it once. It records the stub in
     `mixins` as `Class -> bytes`. A code comment says the instantiation makes
     the class appear in JDI `allClasses()`
     [MixinAgentClassLoader.java:addMixinClass].
   - `addTargetClass(name, classNode)` serializes the pre-mixin `ClassNode`
     with `new ClassWriter(0)` and stores the bytes under `name`
     [MixinAgentClassLoader.java:addTargetClass].
3. **Redefinition.** `Transformer.transform(...)` handles three cases
   [MixinAgent.java:Transformer.transform]:
   - **First load** (`classBeingRedefined == null`): it returns `null`, so the
     class is left unchanged.
   - **Mixin stub**: the class is a stub defined by `addMixinClass`, matched by
     `Class` identity in `mixins`.
     1. It reads the new bytes into a `ClassNode` with `MixinClassReader` and
        `ClassReader.EXPAND_FRAMES`.
     2. It calls `reloadMixin`.
     3. If `reloadMixin` returns `null`, or `reApplyMixins` returns `false`, it
        returns `ERROR_BYTECODE`. Otherwise it returns the stub's bytes
        unchanged.
   - **Any other redefined class**:
     - It logs at INFO "Redefining class {}" and returns
       `classTransformer.transformClassBytes(null, className, classfileBuffer)`.
     - If any `Throwable` is thrown, it logs at ERROR "Error while
       re-transforming class {}" and returns `ERROR_BYTECODE`.
4. **Reload.** `reloadMixin` logs at INFO "Redefining mixin {}". It returns the
   list of target class names from
   `classTransformer.reload(className.replace('/', '.'), classNode)`
   [MixinAgent.java:Transformer.reloadMixin]. It returns `null` on failure:
   - A `MixinReloadException` is logged at ERROR as "Mixin {} cannot be
     reloaded, needs a restart to be applied: {} ".
   - Any other `Throwable` is logged at ERROR as "Error while finding targets
     for mixin {}".
5. **Re-apply.** `reApplyMixins` handles each target in order
   [MixinAgent.java:Transformer.reApplyMixins]:
   1. It converts `/` to `.` in the target name.
   2. It finds the class with
      `MixinService.getService().getClassProvider().findClass`.
   3. It fetches the stored original bytes by the dotted name. If none are
      stored, it logs at ERROR "Target class {} bytecode is not registered" and
      returns `false`.
   4. It transforms the original bytes again with
      `classTransformer.transformClassBytes(null, targetName, originalBytes)`.
   5. It calls `instrumentation.redefineClasses(new ClassDefinition(targetClass, bytes))`.

   If any `Throwable` is thrown, it logs at ERROR "Error while re-transforming
   target class {}" and returns `false`. It returns `true` only when every
   target succeeds.

## Invariants

- The agent acts only when a class is redefined or retransformed. On a class's
  first load, `transform` always returns `null`
  [MixinAgent.java:Transformer.transform].
- A class is treated as a mixin only if its `Class` object is a key of
  `MixinAgentClassLoader.mixins`. A class with the same name from another
  loader is handled as an ordinary class
  [MixinAgentClassLoader.java:getFakeMixinBytecode].
- After a successful mixin reload, `transform` returns the stub's own bytes, so
  the stub class the JVM sees never changes shape. On failure it returns
  `ERROR_BYTECODE`, never `null` [MixinAgent.java:Transformer.transform].
- Targets are always rebuilt from the bytes stored by `addTargetClass`, never
  from the class's current bytes [MixinAgent.java:Transformer.reApplyMixins].
- `reApplyMixins` stops at the first target that fails. Targets it already
  redefined in that pass are not rolled back
  [MixinAgent.java:Transformer.reApplyMixins].
- For a given name, only the first `addTargetClass` call stores bytes. Later
  calls for the same name return early. Every access to `targets` is
  synchronized on the map, while `mixins` is an unsynchronized `HashMap`
  [MixinAgentClassLoader.java:addTargetClass]
  [MixinAgentClassLoader.java:getOriginalTargetBytecode].
- `materialise(name)` generates each stub class
  [MixinAgentClassLoader.java:materialise]:
  - The class is `ACC_PUBLIC`, extends `java/lang/Object` and has no
    interfaces.
  - Its class-file version is
    `MixinEnvironment.getCompatibilityLevel().getClassVersion()`.
  - Its only method is a public no-arg constructor that calls `Object.<init>`.
  - It is written with `ClassWriter.COMPUTE_FRAMES | ClassWriter.COMPUTE_MAXS`.
- `addMixinClass` catches any `Throwable` and logs at ERROR "Catching {}". One
  example is a `LinkageError` from defining the same name twice in one loader.
  No stub is recorded in that case [MixinAgentClassLoader.java:addMixinClass].
- `addTargetClass` catches `Exception`. It logs at ERROR "Error storing original
  class bytecode for {} in mixin hotswap agent. {}: {}", then logs
  `ex.toString()` at DEBUG [MixinAgentClassLoader.java:addTargetClass].
- All agents share one static `MixinAgentClassLoader` and one static
  `Instrumentation` [MixinAgent.java:classLoader] [MixinAgent.java:instrumentation].

## Side Effects

- `premain` sets the JVM system property `mixin.hotSwap` to `"true"`
  [MixinAgent.java:premain].
- `init` and the constructor register `ClassFileTransformer`s with the JVM
  `Instrumentation`. They are never removed [MixinAgent.java:initTransformer].
- The constructor adds `this` to the static `agents` list, which is never
  cleared [MixinAgent.java:MixinAgent(IMixinTransformer)].
- `reApplyMixins` redefines live JVM classes with
  `Instrumentation.redefineClasses`. It can also load target classes through
  the service class provider [MixinAgent.java:Transformer.reApplyMixins].
- `reloadMixin` changes the mixin processor's state through
  `IMixinTransformer.reload` [MixinAgent.java:Transformer.reloadMixin].
- `addMixinClass` defines one class per mixin name in the agent's loader and
  instantiates it once [MixinAgentClassLoader.java:addMixinClass].
- `addTargetClass` keeps a copy of each target's original bytecode for the life
  of the loader. Nothing is evicted [MixinAgentClassLoader.java:addTargetClass].
- Both `log` helpers write to the logger `mixin.agent`
  [MixinAgent.java:log] [MixinAgentClassLoader.java:log]. If `MixinService`
  throws `ServiceNotAvailableError`, they write to `System.err` instead:
  - The output is `"MixinAgent: <LEVEL>: "` followed by
    `String.format(message, params)`, with no trailing newline.
  - `String.format` does not replace `{}` placeholders, so they are printed as
    is.

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileAgentJava`.
- The `agent` source set is declared at build.gradle:105-109. It compiles
  against `main.output` with `languageVersion = 8` and `compatibility = '1.6'`.
- `agentImplementation` extends `implementation` (build.gradle:77).
- The source set is not `modularityExcluded`.
  src/modularity/java/module-info.java exports `org.spongepowered.tools.agent`.
- No tests cover this package at this commit.
