# service

## Overview

The plug-in point for obfuscation types. Each `IObfuscationService` supplies
`ObfuscationTypeDescriptor`s. `ObfuscationServices` finds the services with
`java.util.ServiceLoader` and registers each descriptor as an
`ObfuscationType`. `ObfuscationManager.init` in
`src/ap/java/org/spongepowered/tools/obfuscation/ObfuscationManager.java`
starts this discovery.

## Interface

- `IObfuscationService`: `Set<String> getSupportedOptions()` and
  `Collection<ObfuscationTypeDescriptor> getObfuscationTypes(IMixinAnnotationProcessor ap)`.
  `getObfuscationTypes` may return `null` [IObfuscationService.java:IObfuscationService].
- `ObfuscationServices`: public final singleton with
  `static getInstance()`, `initProviders(IMixinAnnotationProcessor)`,
  `getSupportedOptions()` and `getService(Class<? extends IObfuscationService>)`
  [ObfuscationServices.java:ObfuscationServices].
- `ObfuscationTypeDescriptor` constructors:
  - `(key, inputFileArgName, outFileArgName, Class<? extends ObfuscationEnvironment>)`,
    which sets the extra-input option to `null`.
  - `(key, inputFileArgName, extraInputFilesArgName, outFileArgName, environmentType)`.
  [ObfuscationTypeDescriptor.java:ObfuscationTypeDescriptor]
- `ObfuscationTypeDescriptor` getters: `getKey()` (final),
  `getInputFileOption()`, `getExtraInputFilesOption()`,
  `getOutputFileOption()` and `getEnvironmentType()`
  [ObfuscationTypeDescriptor.java:getKey].
- Registration files, read by `ServiceLoader`:
  - `src/ap/resources/META-INF/services/org.spongepowered.tools.obfuscation.service.IObfuscationService`
    lists `org.spongepowered.tools.obfuscation.mcp.ObfuscationServiceMCP` and
    `org.spongepowered.tools.obfuscation.fg3.ObfuscationServiceFG3`.
  - `src/example/resources/META-INF/services/org.spongepowered.tools.obfuscation.service.IObfuscationService`
    lists `org.spongepowered.tools.obfuscation.json.ObfuscationServiceJson`.

## Invariants

- `getInstance()` creates the singleton lazily without synchronization. The
  constructor calls
  `ServiceLoader.load(IObfuscationService.class, ObfuscationServices.class.getClassLoader())`,
  so services must be visible to the class loader that loaded the processor
  [ObfuscationServices.java:ObfuscationServices].
- `initProviders` runs once per singleton. Calls after the first return
  immediately, even if they pass a different processor instance
  [ObfuscationServices.java:initProviders].
- For each loaded service not already initialized,
  `ObfuscationType.create(descriptor, ap)` runs on every returned descriptor.
  An exception from `create` prints a stack trace and skips that descriptor.
  For example, `create` throws `IllegalArgumentException` for a duplicate key
  [ObfuscationServices.java:initProviders].
- A `ServiceConfigurationError` stops the loop and is reported as
  `Kind.ERROR` with the text `<ErrorSimpleName>: <message>`
  [ObfuscationServices.java:initProviders].
- When at least one type registers, an INFO message lists the types:
  `Supported obfuscation types: <ServiceSimpleName> supports [k1,k2] ...`
  [ObfuscationServices.java:initProviders].
- If no registered type is the default (see `ObfuscationType.isDefault`), a
  `Kind.WARNING` is printed:
  - If `defaultObfuscationEnv` is unset:
    `No default obfuscation environment was specified and "searge" is not available. Please ensure defaultObfuscationEnv is specified in your build configuration`.
  - Otherwise:
    `Specified default obfuscation environment "<value lowercased>" was not defined. This probably means your build configuration is out of date or a required service is missing`.
  [ObfuscationServices.java:initProviders]
- `getSupportedOptions()` returns the union of the options of services that
  `initProviders` has initialized. Before that it returns an empty set
  [ObfuscationServices.java:getSupportedOptions].
- `getService` matches by class name among initialized services and returns
  `null` when none matches [ObfuscationServices.java:getService].
- A descriptor key must be unique. It becomes both the refmap `data` key and
  the `ObfuscationType` key [ObfuscationTypeDescriptor.java:getKey].

## Side Effects

- Iterating the `ServiceLoader` scans the class loader for service files and
  instantiates each listed class [ObfuscationServices.java:initProviders].
- The singleton, its set of initialized services and its `providerInitDone`
  flag are static state that lasts as long as the class loader
  [ObfuscationServices.java:instance].
- `initProviders` adds entries to the static registry in `ObfuscationType`
  [ObfuscationServices.java:initProviders].

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileApJava`
- Run: `mise exec -- ./gradlew test`. Note that `src/test` does not exist yet,
  so no tests cover this package today.
