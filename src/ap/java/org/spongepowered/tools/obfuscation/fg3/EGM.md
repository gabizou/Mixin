# fg3

## Overview

The ForgeGradle 3 obfuscation service. It registers the `searge` obfuscation
type backed by TSRG files, but only when the `mappingTypes` processor option
includes `tsrg`. `ServiceLoader` discovers it through the entry in
`src/ap/resources/META-INF/services/org.spongepowered.tools.obfuscation.service.IObfuscationService`.

## Interface

- `ObfuscationServiceFG3`: public class that implements
  `service.IObfuscationService` [ObfuscationServiceFG3.java:ObfuscationServiceFG3].
- Constants:

  | Constant | Value | Role |
  | --- | --- | --- |
  | `SEARGE` | `"searge"` | Obfuscation type key |
  | `REOBF_TSRG_FILE` | `"reobfTsrgFile"` | Input TSRG file option |
  | `REOBF_EXTRA_TSRG_FILES` | `"reobfTsrgFiles"` | Extra input files option, `;`-separated |
  | `OUT_TSRG_SRG_FILE` | `"outTsrgFile"` | Output TSRG file option |
  | `TSRG_OUTPUT_BEHAVIOUR` | `"mergeBehaviour"` | Output merge option |

  [ObfuscationServiceFG3.java:SEARGE]
- `getSupportedOptions()`: returns the four option keys above, which are every
  constant except `SEARGE` [ObfuscationServiceFG3.java:getSupportedOptions].
- `getObfuscationTypes(IMixinAnnotationProcessor)`: returns the type
  descriptors this service provides [ObfuscationServiceFG3.java:getObfuscationTypes].
- `ObfuscationEnvironmentFG3`: public class that extends `ObfuscationEnvironment`.
  It has a protected constructor `(ObfuscationType)`. It overrides
  `getMappingProvider` and `getMappingWriter` [ObfuscationEnvironmentFG3.java:ObfuscationEnvironmentFG3].

## Invariants

- `getObfuscationTypes` returns one descriptor only when
  `ap.getOptions("mappingTypes")` contains the exact element `"tsrg"`.
  Otherwise it returns an empty list [ObfuscationServiceFG3.java:getObfuscationTypes].
- That descriptor has key `searge`, input `reobfTsrgFile`, extra input
  `reobfTsrgFiles`, output `outTsrgFile` and environment
  `ObfuscationEnvironmentFG3` [ObfuscationServiceFG3.java:getObfuscationTypes].
- `getOptions` splits the option value on `,` and does not trim whitespace. A
  value such as `srg, tsrg` gives the element `" tsrg"`, which does not match
  [AnnotatedMixins.java:getOptions].
- The MCP service registers its `searge` type only when `tsrg` is absent. The
  two services therefore never register the same key in one run (see
  `src/ap/java/org/spongepowered/tools/obfuscation/mcp/ObfuscationServiceMCP.java`).
- The environment reads input with `MappingProviderTSrg` and writes output
  with `MappingWriterTSrg` [ObfuscationEnvironmentFG3.java:getMappingProvider].
- The writer merges the input mappings into the output only when the option
  `mergeBehaviour` equals `merge`, ignoring case. If the option is unset or has
  any other value, the output contains only the generated mappings
  [ObfuscationEnvironmentFG3.java:getMappingWriter].
- `getMappingWriter` passes the provider created by `getMappingProvider` to
  the writer. The base `ObfuscationEnvironment` constructor calls
  `getMappingProvider` first [ObfuscationEnvironmentFG3.java:getMappingWriter].

## Side Effects

- None in this package. The provider and writer it creates read and write
  files (see `src/ap/java/org/spongepowered/tools/obfuscation/mapping/fg3`).

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileApJava`
- Run: `mise exec -- ./gradlew test`. Note that `src/test` does not exist yet,
  so no tests cover this package today.
