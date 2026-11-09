# org.spongepowered.asm.bridge (bridge source set)

## Overview

These adapters let an ASM `org.objectweb.asm.commons.Remapper` serve as two
Mixin interfaces:

- `IRemapper`, in src/main/java/org/spongepowered/asm/mixin/extensibility/IRemapper.java
- `ObfuscationUtil.IClassRemapper`, in src/main/java/org/spongepowered/asm/util/ObfuscationUtil.java

`RemapperAdapter` is the generic base class. `RemapperAdapterFML` wraps Forge's
`FMLDeobfuscatingRemapper` singleton, which it looks up by reflection.

The only caller is outside this partition, in
src/launchwrapper/java/org/spongepowered/asm/launch/platform/MixinPlatformAgentFMLLegacy.java.
That class names the adapter by the string
`"org.spongepowered.asm.bridge.RemapperAdapterFML"` (`FML_REMAPPER_ADAPTER_CLASS`).
Its `injectRemapper` method calls the static `create()` by reflection and adds
the result to `MixinEnvironment.getDefaultEnvironment().getRemappers()`.

## Interface

### RemapperAdapter

- `public abstract class RemapperAdapter implements IRemapper, IClassRemapper`.
  It declares no abstract methods [RemapperAdapter.java:RemapperAdapter].
- `public RemapperAdapter(Remapper remapper)` stores `remapper` without a null
  check [RemapperAdapter.java:RemapperAdapter(Remapper)].
- `protected final Remapper remapper` [RemapperAdapter.java:remapper]
- `protected final ILogger logger`, from
  `MixinService.getService().getLogger("mixin")` [RemapperAdapter.java:logger]
- The methods are:

| Method | Behaviour |
| --- | --- |
| `String mapMethodName(String owner, String name, String desc)` | Returns `remapper.mapMethodName(owner, name, desc)` if that differs from `name`. Otherwise returns `remapper.mapMethodName(unmap(owner), name, unmapDesc(desc))` [RemapperAdapter.java:mapMethodName] |
| `String mapFieldName(String owner, String name, String desc)` | Same two-step lookup, using `remapper.mapFieldName` [RemapperAdapter.java:mapFieldName] |
| `String map(String typeName)` | Returns `remapper.map(typeName)` [RemapperAdapter.java:map] |
| `String unmap(String typeName)` | Returns `typeName` unchanged [RemapperAdapter.java:unmap] |
| `String mapDesc(String desc)` | Returns `remapper.mapDesc(desc)` [RemapperAdapter.java:mapDesc] |
| `String unmapDesc(String desc)` | Returns `ObfuscationUtil.unmapDescriptor(desc, this)`, which applies `this.unmap` to each `L...;` class token [RemapperAdapter.java:unmapDesc] |
| `String toString()` | Returns `getClass().getSimpleName()` [RemapperAdapter.java:toString] |

### RemapperAdapterFML

- `public final class RemapperAdapterFML extends RemapperAdapter`
  [RemapperAdapterFML.java:RemapperAdapterFML]
- `public static IRemapper create()` is the factory, and the only way to
  reach the private constructor. It returns `null` on failure
  [RemapperAdapterFML.java:create].
- `public String unmap(String typeName)` calls the FML remapper's
  `unmap(String)` by reflection and returns `.toString()` of the result
  [RemapperAdapterFML.java:unmap].
- Constants for the class names it looks up
  [RemapperAdapterFML.java:DEOBFUSCATING_REMAPPER_CLASS_FORGE]
  [RemapperAdapterFML.java:DEOBFUSCATING_REMAPPER_CLASS_LEGACY]:
  - `net.minecraftforge.fml.common.asm.transformers.deobf.FMLDeobfuscatingRemapper`
  - `cpw.mods.fml.common.asm.transformers.deobf.FMLDeobfuscatingRemapper`
- Constants for the member names it looks up: `INSTANCE_FIELD = "INSTANCE"` and
  `UNMAP_METHOD = "unmap"`.

## Invariants

- In `mapMethodName` and `mapFieldName`, the fallback lookup runs only when the
  first lookup returns a name that `equals` the input `name`. If the first
  lookup returns `null`, `newName.equals(name)` throws `NullPointerException`
  [RemapperAdapter.java:mapMethodName] [RemapperAdapter.java:mapFieldName].
- In the base class, `unmap` returns its input unchanged. The fallback lookup
  therefore repeats the first lookup with the same owner and descriptor
  [RemapperAdapter.java:unmap].
- `mapDesc` and `unmapDesc` work differently:
  - `mapDesc` uses ASM's `Remapper.mapDesc`.
  - `unmapDesc` uses `ObfuscationUtil.unmapDescriptor`. That helper throws
    `IllegalArgumentException("Invalid descriptor '<desc>', missing ';'")` when
    an `L` token has no closing `;`. When `unmap` returns `null` for a token, it
    keeps the original token. See `remapDescriptor` and `remap` in
    src/main/java/org/spongepowered/asm/util/ObfuscationUtil.java.
- `RemapperAdapterFML.create()` works in three steps
  [RemapperAdapterFML.java:create]
  [RemapperAdapterFML.java:getFMLDeobfuscatingRemapper]:
  1. It loads the Forge class name with `Class.forName`. It tries the legacy
     `cpw.mods` name only if that throws `ClassNotFoundException`.
  2. It reads the static field `INSTANCE` and casts it to `Remapper`.
  3. It looks up the declared method `unmap(String)`.

  It never calls `setAccessible`. If any `Exception` is thrown, it returns
  `null`.
- `RemapperAdapterFML.unmap` never throws an `Exception`. If the reflective call
  throws any `Exception`, or returns `null` so that `.toString()` throws
  `NullPointerException`, it returns `typeName` unchanged
  [RemapperAdapterFML.java:unmap].
- `MixinPlatformAgentFMLLegacy.injectRemapper` does not check the result of
  `create()` for `null`. Because `create()` returns `null` instead of throwing,
  a failed lookup adds `null` to the remapper chain.

## Side Effects

- Constructing any adapter calls `MixinService.getService().getLogger("mixin")`
  [RemapperAdapter.java:logger].
- `mapMethodName`, `mapFieldName` and `map` always log at DEBUG on entry
  [RemapperAdapter.java:mapMethodName] [RemapperAdapter.java:mapFieldName]
  [RemapperAdapter.java:map]:
  - "{} is remapping method {}{} for {}"
  - "{} is remapping field {}{} for {}"
  - "{} is remapping class {}"
- When `mapMethodName` or `mapFieldName` uses the fallback lookup, it logs a
  second DEBUG line: "{} is remapping obfuscated method {}{} for {}" or
  "{} is remapping obfuscated field {}{} for {}".
- `mapDesc`, `unmapDesc` and `unmap` do not log.
- The `RemapperAdapterFML` constructor logs at INFO "Initialised Mixin FML
  Remapper Adapter with {}"
  [RemapperAdapterFML.java:RemapperAdapterFML(Remapper, Method)].
- `create()` loads and initializes the FML remapper class with `Class.forName`
  and reads its static `INSTANCE` field. On failure it calls
  `ex.printStackTrace()`, which writes to `System.err` instead of the Mixin
  logger [RemapperAdapterFML.java:create].
- Each `RemapperAdapterFML.unmap` call makes a reflective `Method.invoke` on the
  FML remapper. Failures are not logged [RemapperAdapterFML.java:unmap].

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileBridgeJava`.
- The `bridge` source set is declared at build.gradle:110-115. It compiles
  against `main.output` with `languageVersion = 8`, `compatibility = '1.8'` and
  `modularityExcluded = true`.
- `bridgeImplementation` does not extend `implementation`, unlike the
  `fernflower` and `agent` configurations (build.gradle:71-82). Its declared
  dependencies are `org.apache.logging.log4j:log4j-core:2.0-beta9` and
  `org.ow2.asm:asm-commons:$legacyForgeAsmVersion` (build.gradle:211-212).
- The ASM version is not the 5.0.3 that the properties file suggests:
  - build.gradle:42 sets `legacyForgeAsmVersion` to `asmVersion` whenever a
    `legacyForgeAsmVersion` property exists.
  - gradle.properties:10 defines that property as `5.0.3`.
  - As a result, the bridge compiles against `asm-commons:9.5`.
    `mise exec -- ./gradlew dependencies --configuration bridgeCompileClasspath`
    confirms this.
- No tests cover this package at this commit.
