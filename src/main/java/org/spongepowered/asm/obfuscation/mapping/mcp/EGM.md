# org.spongepowered.asm.obfuscation.mapping.mcp

## Overview

This package holds MCP-specific mapping structs. Today it has one class, `MappingFieldSrg`, a `MappingField` that keeps the original SRG string and serialises back to it. Its users are in the annotation processor under src/ap/java/org/spongepowered/tools/obfuscation.

## Interface

- `public class MappingFieldSrg extends MappingField` [MappingFieldSrg.java:MappingFieldSrg]:
  - `public MappingFieldSrg(String srg)`
  - `public MappingFieldSrg(MappingField field)`
  - `public String serialise()` [MappingFieldSrg.java:serialise]

## Invariants

- `MappingFieldSrg(String srg)` splits the string at its last `/`. The owner is the part before it, or null when there is no `/`. The name is the part after it, or the whole string. A null `srg` gives a null owner and a null name. The desc is always null [MappingFieldSrg.java:getOwnerFromSrg] [MappingFieldSrg.java:getNameFromSrg].
- `MappingFieldSrg(MappingField field)` copies the owner and name, drops the desc, and stores `srg = owner + "/" + name` [MappingFieldSrg.java:MappingFieldSrg].
- `serialise()` returns the stored `srg` string, not `toString()` [MappingFieldSrg.java:serialise].
- `equals`, `hashCode` and `toString` are inherited from `MappingField`, so they compare `L<owner>;<name>:` and ignore `srg`. The inherited `move`, `remap`, `transform` and `copy` return plain `MappingField` instances, not `MappingFieldSrg` (see the sibling package `mapping/common`).

## Side Effects

- None. The class is immutable and has no static state.

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileJava`
- Run from the repo root: `mise exec -- ./gradlew test`
- `build.gradle` declares a `test` source set, but `src/test` does not exist yet. `test` therefore runs no tests today.
