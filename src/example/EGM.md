# example source set

## Overview

The Gradle `example` source set holds a sample third-party
`IObfuscationService` for the Mixin annotation processor. Its compile
classpath includes the `main` and `ap` outputs. The build marks it with
`modularityExcluded = true`. `stagingJar` excludes it (the filter
`!(it.name =~ /example|test/)`), and the `processor` `shadowJar` contains only
`ap` output, so it does not ship (see `build.gradle`).

The source set also contains `src/example/resources`, which has no code. Its
file `META-INF/services/org.spongepowered.tools.obfuscation.service.IObfuscationService`
registers `org.spongepowered.tools.obfuscation.json.ObfuscationServiceJson`
for `java.util.ServiceLoader`.

## Subdirectories & Boundaries

- java: [./java/EGM.md](./java/EGM.md). Holds the sample service:
  - It registers the obfuscation type `json`.
  - It declares the options `reobfSrgFile` (input, the same option the MCP
    `searge` type uses) and `outputJson` (output).
  - It writes `{"fields": {...}, "methods": {...}}` with Gson pretty
    printing, and writes nothing when `outputJson` is unset.

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileExampleJava`
- Run: `mise exec -- ./gradlew test`. Note that `src/test` does not exist yet.
