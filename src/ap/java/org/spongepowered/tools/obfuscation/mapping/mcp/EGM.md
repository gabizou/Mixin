# mapping/mcp

## Overview

Reads and writes MCP SRG mapping files. `MappingProviderSrg` parses SRG input,
using a format ported from Srg2Source. `MappingWriterSrg` writes the generated
field and method mappings as SRG lines. The FG3 TSRG writer and the MCP
obfuscation environment in `src/ap/java/org/spongepowered/tools/obfuscation/mcp`
both build on these classes. So does the example JSON environment in `src/example`.

## Interface

- `MappingProviderSrg`: public class that extends `mapping.common.MappingProvider`.
  The constructor is `(Messager, Filer)` [MappingProviderSrg.java:MappingProviderSrg].
- `MappingProviderSrg.read(File)`: parses one SRG file and throws `IOException`
  [MappingProviderSrg.java:read].
- `MappingProviderSrg.getFieldMapping(MappingField)`: looks up a field with its
  descriptor removed [MappingProviderSrg.java:getFieldMapping].
- `MappingWriterSrg`: public class that extends `mapping.common.MappingWriter`.
  The constructor is `(Messager, Filer)` [MappingWriterSrg.java:MappingWriterSrg].
- `MappingWriterSrg.write(String output, ObfuscationType, MappingSet<MappingField>, MappingSet<MappingMethod>)`
  [MappingWriterSrg.java:write].
- Protected hooks that subclasses override (as `MappingWriterTSrg` does):
  `openFileWriter(String, ObfuscationType)`, `writeHeader`,
  `writeFieldMappings`, `writeMethodMappings`, `formatFieldMapping` and
  `formatMethodMapping` [MappingWriterSrg.java:formatFieldMapping].

## Invariants

### SRG parsing

- Lines are read with `Files.readLines(input, Charset.defaultCharset(), LineProcessor)`
  [MappingProviderSrg.java:read].
- Empty lines and lines that start with `#` are skipped [MappingProviderSrg.java:read].
- The record type is the first 2 characters. The arguments are the text from
  character 4 onward, split on a single space [MappingProviderSrg.java:read].
- `PK` is a package mapping: `args[0]` maps to `args[1]` [MappingProviderSrg.java:read].
- `CL` is a class mapping: `args[0]` maps to `args[1]` [MappingProviderSrg.java:read].
- `FD` is a field mapping from `owner/name` to `obfOwner/obfName`. Each side is
  parsed with `MappingFieldSrg` (in
  `src/main/java/org/spongepowered/asm/obfuscation/mapping/mcp/MappingFieldSrg.java`)
  and stored without a descriptor [MappingProviderSrg.java:read].
- `MD` is a method mapping from `owner/name desc` to `obfOwner/obfName obfDesc`
  [MappingProviderSrg.java:read].
- Any other record type throws
  `MixinException("Invalid SRG file: " + input)` [MappingProviderSrg.java:read].
- Entries are stored with `BiMap.forcePut`, so a later entry replaces an
  earlier one that has the same key or value [MappingProviderSrg.java:read].

### SRG output

- `write` does nothing when `output` is `null`, so no file is created
  [MappingWriterSrg.java:write].
- The order is: header (empty in this class), then all field lines, then all
  method lines. Each group keeps `MappingSet` insertion order
  [MappingWriterSrg.java:write].
- Field line: `FD: <from.owner>/<from.name> <to.owner>/<to.name>`
  [MappingWriterSrg.java:formatFieldMapping].
- Method line: `MD: <from.name> <from.desc> <to.name> <to.desc>`. Here
  `MappingMethod.getName()` returns `owner/name`
  [MappingWriterSrg.java:formatMethodMapping].
- The file description in log messages is `<type> output SRGs`
  [MappingWriterSrg.java:openFileWriter].
- An `IOException` while writing prints a stack trace and is not reported
  through the messager. Errors when closing the writer are ignored
  [MappingWriterSrg.java:write].

## Side Effects

- `read` reads the input file from disk [MappingProviderSrg.java:read].
- `write` creates the output file through `MappingWriter.openFileWriter`,
  which emits an INFO message [MappingWriterSrg.java:write].

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileApJava`
- Run: `mise exec -- ./gradlew test`. Note that `src/test` does not exist yet,
  so no tests cover this package today.
