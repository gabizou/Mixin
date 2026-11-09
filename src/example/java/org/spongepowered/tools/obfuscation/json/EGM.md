# example: obfuscation/json

## Overview

A sample third-party obfuscation service for the Mixin annotation processor.
It registers an obfuscation type named `json` that reads SRG input with the
processor's `MappingProviderSrg` and writes its generated mappings as a JSON
file. It shows the `IObfuscationService` extension point described in
`src/ap/java/org/spongepowered/tools/obfuscation/service/EGM.md`.
`ServiceLoader` finds it through
`src/example/resources/META-INF/services/org.spongepowered.tools.obfuscation.service.IObfuscationService`.

## Interface

- `ObfuscationServiceJson`: public class that implements `IObfuscationService`.
  Its constants are `KEY = "json"` and `OUTPUT_JSON_FILE = "outputJson"`
  [ObfuscationServiceJson.java:ObfuscationServiceJson].
- `ObfuscationServiceJson.getSupportedOptions()`: returns `reobfSrgFile`, which
  is `ObfuscationServiceMCP.REOBF_SRG_FILE`, and `outputJson`
  [ObfuscationServiceJson.java:getSupportedOptions].
- `ObfuscationServiceJson.getObfuscationTypes(IMixinAnnotationProcessor)`:
  always returns one descriptor with key `json`, input option `reobfSrgFile`,
  no extra-input option, output option `outputJson` and environment
  `ObfuscationEnvironmentJson` [ObfuscationServiceJson.java:getObfuscationTypes].
- `ObfuscationEnvironmentJson`: public class that extends
  `ObfuscationEnvironment`. It has a public constructor `(ObfuscationType)`.
  Its mapping provider is `MappingProviderSrg` and its mapping writer is
  `MappingWriterJson` [ObfuscationEnvironmentJson.java:ObfuscationEnvironmentJson].
- `MappingWriterJson`: public class that extends `mapping.common.MappingWriter`.
  Its methods are `write(String, ObfuscationType, MappingSet<MappingField>, MappingSet<MappingMethod>)`
  and the protected `writeJson(String, ObfuscationType, Map<String, Map<String, String>>)`
  [MappingWriterJson.java:MappingWriterJson].

## Invariants

- The `json` type uses the same input option, `reobfSrgFile`, as the MCP
  `searge` type. Setting that option makes both types supported
  [ObfuscationServiceJson.java:getObfuscationTypes].
- `write` does nothing when the output name is `null`, so no file is written
  when `outputJson` is unset [MappingWriterJson.java:write].
- The output is one JSON object with exactly two keys in this order:
  `"fields"` and `"methods"`. Each maps `from.serialise()` to `to.serialise()`
  and keeps `MappingSet` order [MappingWriterJson.java:write].
- The JSON is produced with `new GsonBuilder().setPrettyPrinting().disableHtmlEscaping()`
  [MappingWriterJson.java:writeJson].
- The output path follows the rule in `MappingWriter.openFileWriter`. A name
  containing `/`, `\` or `:` is a filesystem path, and any other name is a
  `CLASS_OUTPUT` resource. The file description in log messages is
  `<type> output` [MappingWriterJson.java:writeJson].
- The Gradle build excludes the `example` source set from `stagingJar`, and
  `shadowJar` packages only `sourceSets.ap.output`. So this service does not
  ship in the published jars (see `build.gradle`, tasks `stagingJar` and
  `shadowJar`).

## Side Effects

- Creates the output JSON file through `MappingWriter.openFileWriter`, which
  emits an INFO message [MappingWriterJson.java:writeJson].
- An `IOException` while writing prints a stack trace and is not reported
  through the messager. Errors when closing the writer are ignored
  [MappingWriterJson.java:writeJson].
- Reads the SRG input file through `MappingProviderSrg`
  [ObfuscationEnvironmentJson.java:getMappingProvider].

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileExampleJava`
- Run: `mise exec -- ./gradlew test`. Note that `src/test` does not exist yet,
  so no tests cover this package today.
