# mapping/fg3

## Overview

Reads and writes ForgeGradle 3 TSRG mapping files. `MappingProviderTSrg`
parses TSRG input into the base `MappingProvider` tables. `MappingWriterTSrg`
writes the generated mappings in TSRG layout, and can first copy the input
mappings into the output. `MappingMethodLazy` delays remapping a method
descriptor until it is first read. The FG3 obfuscation environment in
`src/ap/java/org/spongepowered/tools/obfuscation/fg3` uses these classes.

## Interface

- `MappingProviderTSrg`: public class that extends `mapping.common.MappingProvider`.
  The constructor is `(Messager, Filer)` [MappingProviderTSrg.java:MappingProviderTSrg].
- `MappingProviderTSrg.read(File)`: parses one TSRG file and throws `IOException`
  [MappingProviderTSrg.java:read].
- `MappingProviderTSrg.getFieldMapping(MappingField)`: looks up a field with its
  descriptor removed [MappingProviderTSrg.java:getFieldMapping].
- `MappingProviderTSrg.getInputMappings()`: package-private. Returns every raw
  line read so far [MappingProviderTSrg.java:getInputMappings].
- `MappingWriterTSrg`: public class that extends `mapping.mcp.MappingWriterSrg`.
  The constructor is `(Messager, Filer, MappingProviderTSrg provider, boolean mergeExisting)`
  [MappingWriterTSrg.java:MappingWriterTSrg].
- `MappingMethodLazy`: public class that extends `MappingMethod`. The
  constructor is `(String owner, String simpleName, String originalDesc, IMappingProvider)`.
  It overrides `getDesc()` and `toString()` [MappingMethodLazy.java:MappingMethodLazy].

## Invariants

### TSRG parsing

- `read` reads lines with `Files.readLines(input, Charset.defaultCharset())`
  and appends them to `inputMappings`. It then parses the whole accumulated
  list, so a second `read` call parses the earlier file's lines again
  [MappingProviderTSrg.java:read].
- The inherited `clear()` does not empty `inputMappings`
  [MappingProviderTSrg.java:inputMappings].
- These lines are skipped: empty lines, and lines that start with `#`, `tsrg2`
  or two tabs [MappingProviderTSrg.java:read].
- Each line is split on a single space [MappingProviderTSrg.java:read].
- A line that starts with one tab is a member of the most recent class line:
  - 2 parts are a field, `name obfName`.
  - 3 parts are a method, `name desc obfName`.
  - Any other count throws
    `IllegalStateException("Error parsing TSRG file, too many arguments: " + line)`.
  - If no class line came before it, the parser throws
    `IllegalStateException("Error parsing TSRG file, found member declaration with no class: " + line)`
  [MappingProviderTSrg.java:read].
- An unindented line with 2 parts is a package or a class:
  - If the first part ends with `/`, it is a package mapping, and the trailing
    `/` is removed from both sides.
  - Otherwise it is a class mapping and becomes the current class
  [MappingProviderTSrg.java:read].
- An unindented line with more than 2 parts is an inline member:
  - 3 parts are a field, `owner name obfName`.
  - 4 parts are a method, `owner name desc obfName`.
  - Any other count throws the "too many arguments" exception.
  - The owner must already be in `classMap`. Otherwise the parser throws
    `IllegalStateException("Error parsing TSRG file, found inline member before class mapping: " + line)`
  [MappingProviderTSrg.java:read].
- An unindented line with one part throws
  `IllegalStateException("Error parsing TSRG, unrecognised directive: " + line)`
  [MappingProviderTSrg.java:read].
- Entries are stored with `BiMap.forcePut`, so a later entry replaces an
  earlier one that has the same key or value [MappingProviderTSrg.java:read].

### Lazy method descriptors

- A method's target is a `MappingMethodLazy` that holds the source descriptor.
  The first `getDesc()` call builds the remapped descriptor and caches it. Each
  `L<name>;` is replaced with the provider's class mapping, and names without a
  mapping are kept unchanged [MappingMethodLazy.java:generateDescriptor].
- The constructor passes `"{" + originalDesc + "}"` to the superclass as the
  descriptor because the superclass uses it to compute the hash
  [MappingMethodLazy.java:MappingMethodLazy].

### TSRG output

- Field line: `<from.owner> <from.simpleName> <to.simpleName>`
  [MappingWriterTSrg.java:formatFieldMapping].
- Method line: `<from.owner> <from.simpleName> <from.desc> <to.simpleName>`
  [MappingWriterTSrg.java:formatMethodMapping].
- When `mergeExisting` is true, the header is every raw input line from the
  provider, before the generated lines [MappingWriterTSrg.java:writeHeader].
- The file description in log messages is `<type> composite mappings`
  [MappingWriterTSrg.java:openFileWriter].
- `write` is inherited from `MappingWriterSrg`. It does nothing when the output
  name is `null`, writes fields before methods, and prints any `IOException`
  stack trace instead of reporting it through the messager
  [MappingWriterSrg.java:write].

## Side Effects

- `read` reads the input file from disk [MappingProviderTSrg.java:read].
- `read` stores every input line in memory for the life of the provider
  [MappingProviderTSrg.java:inputMappings].
- Writing creates the output file through `MappingWriter.openFileWriter`,
  which emits an INFO message [MappingWriterTSrg.java:openFileWriter].

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileApJava`
- Run: `mise exec -- ./gradlew test`. Note that `src/test` does not exist yet,
  so no tests cover this package today.
