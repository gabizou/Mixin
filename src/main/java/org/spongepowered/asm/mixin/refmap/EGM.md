# org.spongepowered.asm.mixin.refmap

## Overview

A reference map (refmap) remaps the string references in mixin annotations to the names used in the running environment. Refmaps are written at compile time by the annotation processor in src/ap and read at runtime by this package. Each mixin config loads its refmap in src/main/java/org/spongepowered/asm/mixin/transformer/MixinConfig.java:

1. If the config has no `referenceMap`, it asks the plugin with `IMixinConfigPlugin.getRefMapperConfig()`. If that also returns null, it uses `ReferenceMapper.DEFAULT_RESOURCE` and suppresses the "could not be read" warning.
2. It calls `ReferenceMapper.read(...)`.
3. If the `mixin.env.remapRefMap` option is on, it wraps the result with `RemappingReferenceMapper.of(env, refMapper)`.
4. `MixinConfig.getReferenceMapper()` returns `ReferenceMapper.DEFAULT_MAPPER` when `mixin.env.disableRefMap` is on. Otherwise it sets the refmap's context to `env.getRefmapObfuscationContext()` and returns it.

## Interface

- `public interface IReferenceMapper` [IReferenceMapper.java:IReferenceMapper]:
  - `boolean isDefault()`, `String getResourceName()`, `String getStatus()`
  - `String getContext()` and `void setContext(String context)`. The context may be null.
  - `String remap(String className, String reference)` and `String remapWithContext(String context, String className, String reference)`
- `public interface IMixinContext` [IMixinContext.java:IMixinContext]:
  - `IMixinInfo getMixin()`, `Extensions getExtensions()`
  - `String getClassName()`, `String getClassRef()`, `String getTargetClassName()`, `String getTargetClassRef()`
  - `IReferenceMapper getReferenceMapper()`, which the javadoc says can be null
  - `boolean getOption(Option)` and `int getPriority()`
- `public final class ReferenceMapper implements IReferenceMapper, Serializable`, with `serialVersionUID = 2L` [ReferenceMapper.java:ReferenceMapper]:
  - `public static final String DEFAULT_RESOURCE = "mixin.refmap.json"` [ReferenceMapper.java:DEFAULT_RESOURCE]
  - `public static final ReferenceMapper DEFAULT_MAPPER`, built with `readOnly = true` and resource `"invalid"` [ReferenceMapper.java:DEFAULT_MAPPER]
  - `public ReferenceMapper()` creates an empty, writable refmap whose resource is `DEFAULT_RESOURCE` [ReferenceMapper.java:ReferenceMapper]
  - `public String addMapping(String context, String className, String reference, String newReference)` [ReferenceMapper.java:addMapping]
  - `public void write(Appendable writer)` [ReferenceMapper.java:write]
  - `public static ReferenceMapper read(String resourcePath)` and `public static ReferenceMapper read(Reader reader, String name)` [ReferenceMapper.java:read]
- `public final class RemappingReferenceMapper implements IReferenceMapper`. Its only public factory is `public static IReferenceMapper of(MixinEnvironment env, IReferenceMapper refMap)` [RemappingReferenceMapper.java:of]

## Invariants

### JSON format and defaults

- Gson (`com.google.gson`) reads and writes refmaps. The serialized fields are `mappings` and `data`. `mappings` has the shape `{ "<mixin class ref>": { "<reference>": "<remapped>" } }`. `data` has the shape `{ "<context>": { "<mixin class ref>": { "<reference>": "<remapped>" } } }`. The fields `readOnly`, `context` and `resource` are `transient` and are never serialized [ReferenceMapper.java:mappings] [ReferenceMapper.java:data].
- `write` uses `GsonBuilder().setPrettyPrinting().disableHtmlEscaping()` [ReferenceMapper.java:write].
- `isDefault()` returns the `readOnly` flag. Only `DEFAULT_MAPPER` is read-only, because the read-only constructor is private. The javadoc tells callers to use `isDefault()` instead of comparing references with `DEFAULT_MAPPER`, "because of classloader shenanigans" [ReferenceMapper.java:isDefault] [IReferenceMapper.java:isDefault].
- `getStatus()` returns `"No refMap loaded."` for a default mapper and `"Using refmap <resource>"` otherwise [ReferenceMapper.java:getStatus].
- The private `setResourceName` does nothing on a read-only mapper. A null name is stored as `"<unknown resource>"` [ReferenceMapper.java:setResourceName].

### Lookup

- `remap(className, reference)` is `remapWithContext(this.context, className, reference)` [ReferenceMapper.java:remap].
- `remapWithContext` uses `mappings` when the context is null. When the context is set but has no entry in `data`, it also falls back to `mappings` [ReferenceMapper.java:remapWithContext].
- With a null `className`, the first class map, in `HashMap` order, that contains `reference` wins [ReferenceMapper.java:remap].
- If there is no map for the class, or no entry for `reference`, the original `reference` is returned unchanged [ReferenceMapper.java:remap].

### Adding mappings

- `addMapping` returns null and changes nothing when the mapper is read-only or when `reference` or `newReference` is null [ReferenceMapper.java:addMapping].
- It removes all whitespace (`\s`) from `reference`. If the stripped reference equals `newReference`, it returns null and changes nothing [ReferenceMapper.java:addMapping].
- A null `context` writes to `mappings`. A non-null context creates its `data` entry if needed. The return value is the previous value, as `Map.put` returns it [ReferenceMapper.java:addMapping].

### Reading

- `read(String)` opens the resource with `MixinService.getService().getResourceAsStream(resourcePath)` and wraps it in `new InputStreamReader(resource)`, which uses the platform default charset [ReferenceMapper.java:read].
- `read(String)` returns `DEFAULT_MAPPER` in three cases:
  - The stream is null. Nothing is logged.
  - Gson throws `JsonParseException`. The error is `Invalid REFMAP JSON in <path>: <exception class> <message>`.
  - Any other `Exception` is thrown. The error is `Failed reading REFMAP JSON from <path>: <exception class> <message>`.
- `read(String)` closes the reader quietly in all cases [ReferenceMapper.java:read].
- `read(Reader, String)` returns `DEFAULT_MAPPER` on any `Exception`, logs nothing and does not close the reader [ReferenceMapper.java:read].

### RemappingReferenceMapper

- `of(env, refMap)` returns `refMap` unchanged when `refMap.isDefault()` is true or when the SRG file name is null or the file does not exist [RemappingReferenceMapper.java:of] [RemappingReferenceMapper.java:hasData].
- The SRG file comes from the `REFMAP_REMAP_RESOURCE` option (`mixin.env.refMapRemappingFile`). When that is empty, it comes from the system property `net.minecraftforge.gradle.GradleStart.srg.srg-mcp` [RemappingReferenceMapper.java:getResource] [RemappingReferenceMapper.java:DEFAULT_RESOURCE_PATH_PROPERTY].
- The constructor sets the inner refmap's context to the `REFMAP_REMAP_SOURCE_ENV` option (`mixin.env.refMapRemappingEnv`), or to `"searge"` when that is empty [RemappingReferenceMapper.java:getMappingEnv].
- `setContext` does nothing. `getContext`, `isDefault`, `getResourceName` and `getStatus` delegate to the inner refmap [RemappingReferenceMapper.java:setContext].
- `remap(className, reference)` first checks a per-instance cache keyed by `className` and then `reference`. On a miss it calls `inner.remap(...)`, applies every SRG entry as a plain `String.replace` in `HashMap` order, and caches the result [RemappingReferenceMapper.java:remap].
- `remapWithContext` delegates straight to the inner refmap, with no SRG replacement and no caching [RemappingReferenceMapper.java:remapWithContext].
- SRG parsing [RemappingReferenceMapper.java:loadSrgs]:
  - Only lines that start with `FD: ` or `MD: ` are used. Blank lines and lines starting with `#` are skipped.
  - The key is the part after the last `/` of the first token. For `FD`, the value is the same part of the second token. For `MD`, it is the same part of the third token.
  - An empty map is put in the static cache before the file is read. A missing file is therefore cached as empty for the life of the class.

## Side Effects

- Static state:
  - `ReferenceMapper.DEFAULT_MAPPER` is one shared instance, and its `setContext` is not guarded. Setting a context on it changes global state [ReferenceMapper.java:setContext].
  - `RemappingReferenceMapper.srgs` is an unsynchronized static cache keyed by SRG file name, and it is never cleared [RemappingReferenceMapper.java:srgs].
- Class initialization of `RemappingReferenceMapper` calls `MixinService.getService().getLogger("mixin")`, which starts the service lookup if it has not run yet [RemappingReferenceMapper.java:logger].
- I/O:
  - `ReferenceMapper.read(String)` reads a classpath resource through the active `IMixinService` [ReferenceMapper.java:read].
  - `RemappingReferenceMapper` reads the SRG file from disk as UTF-8 [RemappingReferenceMapper.java:loadSrgs].
- Logging:
  - `ReferenceMapper.read(String)` reports parse and read failures through `MessageRouter.getMessager()` at `Kind.ERROR` [ReferenceMapper.java:read].
  - `RemappingReferenceMapper` logs `Remapping refMap {} using {}` at info, and logs `Could not read input SRG file: {}` plus `catching(ex)` when an `IOException` occurs [RemappingReferenceMapper.java:RemappingReferenceMapper] [RemappingReferenceMapper.java:loadSrgs].
- Launcher note: src/launchwrapper/java/org/spongepowered/asm/service/mojang/MixinServiceLaunchWrapper.java sets the system property `mixin.env.remapRefMap=true` in `getInitialPhase()` when `sun.java.command` contains `GradleStart`.

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileJava`
- Run from the repo root: `mise exec -- ./gradlew test`
- `build.gradle` declares a `test` source set, but `src/test` does not exist yet. `test` therefore runs no tests today.
