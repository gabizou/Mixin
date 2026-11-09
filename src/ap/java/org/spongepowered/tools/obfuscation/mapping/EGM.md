# mapping

## Overview

The interfaces for mapping input (providers), mapping collection (consumers)
and mapping output (writers), plus the ordered pair set that carries the
generated mappings. The subpackages contain the shared base classes and the
SRG and TSRG implementations.

## Interface

- `IMappingProvider`: `clear()`, `isEmpty()`, `read(File) throws IOException`,
  `getMethodMapping(MappingMethod)`, `getFieldMapping(MappingField)`,
  `getClassMapping(String)` and `getPackageMapping(String)`
  [IMappingProvider.java:IMappingProvider].
- `IMappingConsumer`: `clear()`, `addFieldMapping(ObfuscationType, MappingField from, MappingField to)`,
  `addMethodMapping(ObfuscationType, MappingMethod from, MappingMethod to)`,
  `getFieldMappings(ObfuscationType)` and `getMethodMappings(ObfuscationType)`
  [IMappingConsumer.java:IMappingConsumer].
- `IMappingConsumer.MappingSet<TMapping>`: public class that extends
  `LinkedHashSet<MappingSet.Pair<TMapping>>` [IMappingConsumer.java:MappingSet].
- `IMappingConsumer.MappingSet.Pair<TMapping>`: has public final fields `from`
  and `to` [IMappingConsumer.java:Pair].
- `IMappingWriter`: `write(String output, ObfuscationType type, MappingSet<MappingField> fields, MappingSet<MappingMethod> methods)`
  [IMappingWriter.java:write].
- Subpackages:
  - [common](./common/EGM.md): `MappingProvider` and `MappingWriter` base
    classes, including the rule that decides between a filesystem path and
    `CLASS_OUTPUT`.
  - [fg3](./fg3/EGM.md): TSRG provider and writer, and `MappingMethodLazy`.
  - [mcp](./mcp/EGM.md): SRG provider and writer.

## Invariants

- `Pair.equals` and `hashCode` compare both `from` and `to`. A `MappingSet`
  therefore drops repeated identical pairs and keeps insertion order.
  `Pair.toString()` returns `from -> to` [IMappingConsumer.java:Pair].
- Provider lookups return `null` when no mapping exists [IMappingProvider.java:getMethodMapping].
- The environment calls `read` once for each input file the user supplies.
  The contract says to call `clear()` before reading [IMappingProvider.java:read].
- A provider supplies plain lookups only. Searching superclasses is the job of
  the environment layer [IMappingProvider.java:IMappingProvider].

## Side Effects

- None in this package, which contains only interfaces and the pair set. The
  implementations in `common`, `fg3` and `mcp` read and write files.

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileApJava`
- Run: `mise exec -- ./gradlew test`. Note that `src/test` does not exist yet,
  so no tests cover this package today.
