# mcp

## Overview

The default MCP obfuscation service. It registers the SRG-backed `searge` and
`notch` obfuscation types. `ServiceLoader` discovers it through the first
entry in
`src/ap/resources/META-INF/services/org.spongepowered.tools.obfuscation.service.IObfuscationService`.

## Interface

- `ObfuscationServiceMCP`: public class that implements
  `service.IObfuscationService` [ObfuscationServiceMCP.java:ObfuscationServiceMCP].
- Constants:

  | Constant | Value | Role |
  | --- | --- | --- |
  | `SEARGE` | `"searge"` | Obfuscation type key |
  | `NOTCH` | `"notch"` | Obfuscation type key |
  | `REOBF_SRG_FILE` | `"reobfSrgFile"` | `searge` input SRG option |
  | `REOBF_EXTRA_SRG_FILES` | `"reobfSrgFiles"` | `searge` extra inputs, `;`-separated |
  | `REOBF_NOTCH_FILE` | `"reobfNotchSrgFile"` | `notch` input SRG option |
  | `REOBF_EXTRA_NOTCH_FILES` | `"reobfNotchSrgFiles"` | `notch` extra inputs, `;`-separated |
  | `OUT_SRG_SRG_FILE` | `"outSrgFile"` | `searge` output SRG option |
  | `OUT_NOTCH_SRG_FILE` | `"outNotchSrgFile"` | `notch` output SRG option |

  [ObfuscationServiceMCP.java:SEARGE]
- `getSupportedOptions()`: returns the six option keys, which are every
  constant except `SEARGE` and `NOTCH` [ObfuscationServiceMCP.java:getSupportedOptions].
- `getObfuscationTypes(IMixinAnnotationProcessor)`: returns the type
  descriptors this service provides [ObfuscationServiceMCP.java:getObfuscationTypes].
- `ObfuscationEnvironmentMCP`: public class that extends `ObfuscationEnvironment`.
  It has a protected constructor `(ObfuscationType)`. Its mapping provider is
  `MappingProviderSrg` and its mapping writer is `MappingWriterSrg`
  [ObfuscationEnvironmentMCP.java:getMappingProvider].

## Invariants

- The `searge` descriptor (`reobfSrgFile`, `reobfSrgFiles`, `outSrgFile`) is
  included only when `ap.getOptions("mappingTypes")` does not contain the
  exact element `"tsrg"` [ObfuscationServiceMCP.java:getObfuscationTypes].
- The `notch` descriptor (`reobfNotchSrgFile`, `reobfNotchSrgFiles`,
  `outNotchSrgFile`) is always included [ObfuscationServiceMCP.java:getObfuscationTypes].
- Both descriptors use `ObfuscationEnvironmentMCP` [ObfuscationServiceMCP.java:getObfuscationTypes].
- A returned type gets an environment only if at least one of its input-file
  options is set. See `ObfuscationType.isSupported` in
  `src/ap/java/org/spongepowered/tools/obfuscation/ObfuscationType.java`.

## Side Effects

- None in this package. The SRG provider and writer it creates read and write
  files (see `src/ap/java/org/spongepowered/tools/obfuscation/mapping/mcp`).

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileApJava`
- Run: `mise exec -- ./gradlew test`. Note that `src/test` does not exist yet,
  so no tests cover this package today.
