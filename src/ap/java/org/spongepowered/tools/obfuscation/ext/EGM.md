# ext

## Overview

Holds `SpecialPackages`, a global registry of package prefixes. When the
annotation processor cannot find a mapping for a member in one of these
packages, it does not warn. The registry does not change how mapping lookups
work [SpecialPackages.java:addExcludedPackage].

## Interface

- `SpecialPackages`: public final class with a private constructor and static
  methods only [SpecialPackages.java:SpecialPackages].
- `addExcludedPackage(String packageName)`: public static final
  [SpecialPackages.java:addExcludedPackage].
- `isExcludedPackage(String internalName)`: public static boolean
  [SpecialPackages.java:isExcludedPackage].

## Invariants

- The default prefixes are added in a static initializer: `java/`, `javax/`,
  `sun/` and `com/sun/` [SpecialPackages.java:static].
- `addExcludedPackage` replaces `.` with `/` and adds a trailing `/` if there
  is none [SpecialPackages.java:addExcludedPackage].
- `isExcludedPackage` returns true when `internalName` starts with any
  registered prefix. The name must use `/` separators
  [SpecialPackages.java:isExcludedPackage].
- The injector handler in
  `src/ap/java/org/spongepowered/tools/obfuscation/AnnotatedMixinElementHandlerInjector.java`
  checks this registry before it raises `NO_OBFDATA_FOR_CLASS`,
  `NO_OBFDATA_FOR_FIELD` or `NO_OBFDATA_FOR_METHOD`.

## Side Effects

- `addExcludedPackage` changes a static, unsynchronized `HashSet`. The set
  lasts as long as the class loader, and nothing removes entries from it
  [SpecialPackages.java:suppressWarningsForPackages].

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileApJava`
- Run: `mise exec -- ./gradlew test`. Note that `src/test` does not exist yet,
  so no tests cover this package today.
