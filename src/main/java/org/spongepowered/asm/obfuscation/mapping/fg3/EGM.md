# org.spongepowered.asm.obfuscation.mapping.fg3

## Overview

In `src/main`, this package contains only `package-info.java`. Its javadoc describes the package as "Structs for method and field mappings with specific behaviour for ForgeGradle 3 and above" [package-info.java:org.spongepowered.asm.obfuscation.mapping.fg3]. The main source set declares no classes in it. The annotation processor source set has a separate package, src/ap/java/org/spongepowered/tools/obfuscation/mapping/fg3, which contains FG3-specific mapping code.

## Interface

- `package-info.java` declares the package and its javadoc, and nothing else [package-info.java:org.spongepowered.asm.obfuscation.mapping.fg3].

## Invariants

- None.

## Side Effects

- None.

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileJava`
- Run from the repo root: `mise exec -- ./gradlew test`
- `build.gradle` declares a `test` source set, but `src/test` does not exist yet. `test` therefore runs no tests today.
