# interfaces

## Overview

The service contracts between the parts of the Mixin annotation processor:
messaging, options, obfuscation data lookups, the refmap, environments, type
handles, validators and javadoc access. It also defines two enums whose values
users can observe: `IMessagerEx.MessageType`, which sets the severity of every
diagnostic, and `IMixinAnnotationProcessor.CompilerEnvironment`, which detects
the IDE. The package-private `AnnotatedMixins` in
`src/ap/java/org/spongepowered/tools/obfuscation/AnnotatedMixins.java`
implements most of these interfaces.

## Interface

- `IMessagerEx extends javax.annotation.processing.Messager`: adds four
  `printMessage(MessageType, CharSequence[, Element[, AnnotationMirror[, AnnotationValue]]])`
  overloads [IMessagerEx.java:IMessagerEx].
- `IMessagerEx.MessageType`: public static enum with instance methods
  `isError()`, `getKind()`, `setKind(Kind)`, `quench(Kind)`, `isEnabled()`,
  `setEnabled(boolean)`, `reset()` and `decorate(CharSequence)`. Its static
  methods are `setDecoration(boolean)`, `setPrefix(String)`,
  `getSupportedOptions()` and `applyOptions(CompilerEnvironment, IOptionProvider)`
  [IMessagerEx.java:MessageType].
- `IMessagerSuppressible extends IMessagerEx`: adds `printMessage` overloads
  that take a final `SuppressedBy` argument, for both `Diagnostic.Kind` and
  `MessageType` [IMessagerSuppressible.java:IMessagerSuppressible].
- `IMixinAnnotationProcessor extends IMessagerSuppressible, IOptionProvider`:
  `getCompilerEnvironment()`, `getProcessingEnvironment()`,
  `getObfuscationManager()`, `getTokenProvider()`, `getTypeProvider()` and
  `getJavadocProvider()` [IMixinAnnotationProcessor.java:IMixinAnnotationProcessor].
- `IMixinAnnotationProcessor.CompilerEnvironment`: enum with values `JAVAC`,
  `JDT` and `IDEA`. Methods are `isCompiler()`, `isDevelopmentEnvironment()`,
  `getFriendlyName()` and the static `detect(ProcessingEnvironment)`
  [IMixinAnnotationProcessor.java:CompilerEnvironment].
- `IMixinValidator`: `validate(ValidationPass, TypeElement, IAnnotationHandle, Collection<TypeHandle>)`.
  The nested enum `ValidationPass` has values `EARLY`, `LATE` and `FINAL`
  [IMixinValidator.java:ValidationPass].
- `IOptionProvider`: `getOption(String)`, `getOption(String, String)`,
  `getOption(String, boolean)` and `getOptions(String)`, which reads a
  comma-separated option [IOptionProvider.java:IOptionProvider].
- `IObfuscationManager`: `init()`, `getDataProvider()`,
  `getReferenceManager()`, `createMappingConsumer()`, `getEnvironments()`,
  `writeMappings()` and `writeReferences()` [IObfuscationManager.java:IObfuscationManager].
- `IObfuscationDataProvider`:
  - `getObfEntryRecursive`, `getObfEntry(ITargetSelectorRemappable)` and
    `getObfEntry(IMapping)`.
  - `getObfMethodRecursive`, `getObfMethod` (two overloads) and
    `getRemappedMethod` (two overloads).
  - `getObfFieldRecursive` and `getObfField` (two overloads).
  - `getObfClass(TypeHandle)` and `getObfClass(String)`.
  - Every method returns `ObfuscationData`.
  [IObfuscationDataProvider.java:IObfuscationDataProvider]
- `IObfuscationEnvironment`: `getObfMethod` (three overloads, one of which
  takes `boolean lazyRemap`), `getObfField` (three overloads), `getObfClass`,
  `remapDescriptor(ITargetSelectorRemappable)`, `remapDescriptor(String)` and
  `writeMappings(Collection<IMappingConsumer>)` [IObfuscationEnvironment.java:IObfuscationEnvironment].
- `IReferenceManager`: `setAllowConflicts`, `getAllowConflicts`, `write()`,
  `getMapper()`, `addMethodMapping` (two overloads), `addFieldMapping` and
  `addClassMapping` [IReferenceManager.java:IReferenceManager].
- `ITypeHandleProvider`: `getTypeHandle(String)`, `getTypeHandle(Object)` and
  `getSimulatedHandle(String, TypeMirror)` [ITypeHandleProvider.java:ITypeHandleProvider].
- `IJavadocProvider`: `getJavadoc(Element)` [IJavadocProvider.java:IJavadocProvider].

### MessageType defaults

Every constant starts enabled with the kind shown. `applyOptions` then changes
`INFO` and `INJECTOR_IN_INTERFACE` (see Invariants) [IMessagerEx.java:MessageType].

| Default kind | Constants |
| --- | --- |
| `NOTE` | `INFO`, `NOTE` |
| `OTHER` | `MISSING_INJECTOR_DESC_SIMULATED` |
| `WARNING` | `WARNING`, `MIXIN_SOFT_TARGET_NOT_RESOLVED`, `MIXIN_SOFT_TARGET_IS_PUBLIC`, `ACCESSOR_TYPE_UNSUPPORTED`, `ACCESSOR_NAME_UNRESOLVED`, `INVOKER_RAW_RETURN_TYPE`, `INVALID_CONSTRAINT`, `INJECTOR_ON_NON_METHOD_ELEMENT`, `SOFT_IMPLEMENTS_EMPTY`, `MISSING_INJECTOR_DESC_SINGLETARGET`, `TARGET_ELEMENT_NOT_FOUND`, `METHOD_VISIBILITY`, `NO_OBFDATA_FOR_ACCESSOR`, `NO_OBFDATA_FOR_CLASS`, `NO_OBFDATA_FOR_CTOR`, `NO_OBFDATA_FOR_STATIC_OVERWRITE`, `NO_OBFDATA_FOR_FIELD`, `NO_OBFDATA_FOR_METHOD`, `NO_OBFDATA_FOR_SHADOW`, `NO_OBFDATA_FOR_SIMULATED_SHADOW`, `BARE_REFERENCE`, `OVERWRITE_DOCS` |
| `ERROR` | `ERROR`, `MIXIN_ON_INVALID_TYPE`, `MIXIN_SOFT_TARGET_NOT_FOUND`, `MIXIN_NO_TARGETS`, `PARENT_VALIDATOR`, `TARGET_VALIDATOR`, `ACCESSOR_ATTACH_ERROR`, `ACCESSOR_TARGET_NOT_FOUND`, `FACTORY_INVOKER_GENERIC_ARGS`, `FACTORY_INVOKER_RETURN_TYPE`, `FACTORY_INVOKER_NONSTATIC`, `CONSTRAINT_VIOLATION`, `ACCESSOR_MAPPING_CONFLICT`, `INJECTOR_MAPPING_CONFLICT`, `OVERWRITE_MAPPING_CONFLICT`, `SHADOW_MAPPING_CONFLICT`, `INJECTOR_IN_INTERFACE`, `OVERWRITE_ON_NON_METHOD_ELEMENT`, `ACCESSOR_ON_NON_METHOD_ELEMENT`, `SHADOW_ON_INVALID_ELEMENT`, `INJECTOR_ON_NON_MIXIN_METHOD`, `OVERWRITE_ON_NON_MIXIN_METHOD`, `ACCESSOR_ON_NON_MIXIN_METHOD`, `SHADOW_ON_NON_MIXIN_ELEMENT`, `SOFT_IMPLEMENTS_ON_INVALID_TYPE`, `SOFT_IMPLEMENTS_ON_NON_MIXIN`, `TARGET_SELECTOR_VALIDATION`, `INJECTOR_TARGET_NOT_FULLY_QUALIFIED`, `MISSING_INJECTOR_DESC_MULTITARGET`, `NO_OBFDATA_FOR_TARGET`, `NO_OBFDATA_FOR_OVERWRITE`, `NO_OBFDATA_FOR_SOFT_IMPLEMENTS` |

No code in the `ap` or `example` source sets emits `FACTORY_INVOKER_GENERIC_ARGS`
or `INVOKER_RAW_RETURN_TYPE`. Tests cannot trigger them through this processor.

## Invariants

### MessageType options

- Every constant has a processor option named `MSG_<CONSTANT_NAME>`, built
  from the private prefix `OPTION_PREFIX = "MSG_"`. `getSupportedOptions()`
  returns all of these names [IMessagerEx.java:getSupportedOptions].
- `applyOptions` applies these rules in this order [IMessagerEx.java:applyOptions]:
  1. `showMessageTypes` equal to `true` (ignoring case) turns decoration on.
     Any other value turns it off.
  2. `INFO` is enabled unless the environment is an IDE or `quiet` equals
     `true` (ignoring case).
  3. `INJECTOR_IN_INTERFACE` is enabled only when
     `Boolean.parseBoolean(disableInterfaceMixins)` is true. It is therefore
     disabled by default.
  4. `overwriteErrorLevel` equal to `error` (ignoring case) sets
     `OVERWRITE_DOCS` to `ERROR`.
  5. For each constant, `MSG_<NAME>` set to `note`, `warning` or `error`
     (ignoring case) sets the kind. `disabled` disables the constant. Other
     values are ignored.
- `setKind` and `setEnabled` mark a constant as set by the user. `quench(kind)`
  changes the kind only when the user has not set it and
  `kind.ordinal() > current.ordinal()`, which means a less severe
  `Diagnostic.Kind` [IMessagerEx.java:quench].
- `decorate(msg)` returns `prefix + "[" + NAME + "] " + msg` when decoration is
  on, and `prefix + msg` otherwise. The prefix defaults to `""`
  [IMessagerEx.java:decorate].
- `isError()` is true only when the current kind is `Kind.ERROR`
  [IMessagerEx.java:isError].

### CompilerEnvironment detection

- `detect` checks the values in declaration order and returns the first
  match. It falls back to `JAVAC`, which never matches itself
  [IMixinAnnotationProcessor.java:detect].
- `JDT` matches when the class name of the `ProcessingEnvironment` contains
  `jdt` [IMixinAnnotationProcessor.java:JDT].
- `IDEA` matches when any of these system properties is set:
  `idea.plugins.path`, `idea.config.path`, `idea.home.path` or
  `idea.paths.selector` [IMixinAnnotationProcessor.java:IDEA].
- `JDT` and `IDEA` report `isDevelopmentEnvironment() == true`, and `JAVAC`
  reports false. Their friendly names are `Java Compiler`, `Eclipse (JDT)`
  and `IntelliJ IDEA` [IMixinAnnotationProcessor.java:CompilerEnvironment].

### Other contracts

- A validator returns false from `validate` to stop the validators after it
  from running [IMixinValidator.java:validate].
- `IObfuscationEnvironment` lookups return `null` when no mapping exists. With
  `lazyRemap == false`, a lookup may still remap the owner and descriptor when
  the name has no mapping [IObfuscationEnvironment.java:getObfMethod].
- By default, `IReferenceManager` raises `ReferenceConflictException` when a
  different value is added for an existing key [IReferenceManager.java:getAllowConflicts].

## Side Effects

- `MessageType` keeps mutable static state: each constant's `kind`, `enabled`
  and `setByUser` fields, plus the shared `decorate` flag and `prefix`. This
  state lasts as long as the class loader. `applyOptions`, `setKind`,
  `setEnabled`, `quench`, `setDecoration` and `setPrefix` change it
  [IMessagerEx.java:MessageType].
- No code in the `ap` or `example` source sets calls `MessageType.reset()`. So
  kinds set during one compilation carry over to later compilations in the
  same class loader [IMessagerEx.java:reset].
- `CompilerEnvironment.IDEA` detection reads system properties
  [IMixinAnnotationProcessor.java:IDEA].

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileApJava`
- Run: `mise exec -- ./gradlew test`. Note that `src/test` does not exist yet,
  so no tests cover this package today.
