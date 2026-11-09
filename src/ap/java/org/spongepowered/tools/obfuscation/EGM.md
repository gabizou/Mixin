# org.spongepowered.tools.obfuscation (Mixin annotation processor)

## Overview

The base package of the Mixin annotation processor (AP). Two
`javax.annotation.processing` processors find `@Mixin` classes and their
annotated members during compilation, and check them against target classes
found through mirror, ASM or simulation. They then:

- look up obfuscated names in the loaded mapping environments;
- record the remapped references in a refmap (`ReferenceMapper` JSON); and
- write generated mapping files (SRG or TSRG) for each obfuscation type.

Shared state for one compilation lives in the package-private `AnnotatedMixins`
singleton, one for each `ProcessingEnvironment`.

## Interface

### Entry points

- Processors registered in
  `src/ap/resources/META-INF/services/javax.annotation.processing.Processor`:
  - `MixinObfuscationProcessorInjection` (public). Its supported annotation
    types are `org.spongepowered.asm.mixin.injection.At` plus every class
    returned by `InjectionInfo.getRegisteredAnnotations()`
    [MixinObfuscationProcessorInjection.java:getSupportedAnnotationTypes].
    In `src/main/java/org/spongepowered/asm/mixin/injection/struct/InjectionInfo.java`
    those are `@Inject`, `@ModifyArg`, `@ModifyArgs`, `@Redirect`,
    `@ModifyVariable` and `@ModifyConstant`.
  - `MixinObfuscationProcessorTargets` (public). Its
    `@SupportedAnnotationTypes` lists `org.spongepowered.asm.mixin.Mixin`,
    `Shadow`, `Overwrite`, `gen.Accessor` and `Implements`. `@Invoker` is not
    listed, but `process` also handles it
    [MixinObfuscationProcessorTargets.java:processInvokers].
- `MixinObfuscationProcessor` (package-private, abstract) is the base class of
  both processors:
  - `getSupportedSourceVersion()` returns `SourceVersion.latestSupported()`.
  - `getSupportedOptions()` returns `SupportedOptions.getAllOptions()`.
  - `init` fetches the shared `AnnotatedMixins` for the environment.
  [MixinObfuscationProcessor.java:MixinObfuscationProcessor]
- `SupportedOptions` (public final) holds the option-key constants and
  `getAllOptions()` [SupportedOptions.java:getAllOptions].
- `SuppressedBy` (public enum) holds the `@SuppressWarnings` tokens, available
  through `getToken()` [SuppressedBy.java:SuppressedBy].

### Model and services

- `ObfuscationType` (public final):
  - Constant `DEFAULT_TYPE = "searge"`.
  - Static methods: `create(descriptor, ap)`, `get(key)` and `types()`.
  - Instance methods: `getKey`, `getConfig`, `getAnnotationProcessor`,
    `isDefault`, `isSupported`, `getInputFileNames`, `getOutputFileName` and
    `createEnvironment`.
  [ObfuscationType.java:ObfuscationType]
- `ObfuscationEnvironment` (public abstract, implements `IObfuscationEnvironment`).
  Subclasses implement `getMappingProvider(Messager, Filer)` and
  `getMappingWriter(Messager, Filer)`. Other methods are `getType()`,
  `getObfMethod`, `getObfField`, `getObfClass`, `remapDescriptor` and
  `writeMappings` [ObfuscationEnvironment.java:ObfuscationEnvironment].
- `ObfuscationManager` (public, implements `IObfuscationManager`) owns the
  environments, the data provider, the reference manager and the mapping
  consumers [ObfuscationManager.java:ObfuscationManager].
- `ObfuscationDataProvider` (public, implements `IObfuscationDataProvider`)
  runs each lookup against every environment [ObfuscationDataProvider.java:ObfuscationDataProvider].
- `ObfuscationData<T>` (public, `Iterable<ObfuscationType>`) is a map from
  obfuscation type to value with an optional default. Its methods are `put`,
  the deprecated `add`, `get`, `isEmpty` and `values` [ObfuscationData.java:ObfuscationData].
- `ReferenceManager` (public, implements `IReferenceManager`) builds and writes
  the refmap. It throws the public nested `ReferenceConflictException`, which
  has `getOld()` and `getNew()` [ReferenceManager.java:ReferenceManager].
- `TargetMap` (public final, `HashMap<TypeReference, Set<TypeReference>>`) is
  a serializable map from each target to its mixins. Its methods are
  `getSessionId`, `readImports(File)`, `write(boolean)` and the static
  `create(String)` [TargetMap.java:TargetMap].
- `MixinValidator` (public abstract, implements `IMixinValidator`) is the base
  class for the validators in `validation`. `validate(pass, ...)` does nothing
  for any pass other than its own [MixinValidator.java:validate].

### Package-private classes

- `AnnotatedMixins`: implements `IMixinAnnotationProcessor`, `ITokenProvider`,
  `ITypeHandleProvider` and `IJavadocProvider`. Instances come from
  `getMixinsForEnvironment(ProcessingEnvironment)` [AnnotatedMixins.java:getMixinsForEnvironment].
- `AnnotatedMixin`: the state for one mixin class [AnnotatedMixin.java:AnnotatedMixin].
- `AnnotatedMixinElementHandler` and its subclasses `...Accessor`,
  `...Injector`, `...Overwrite`, `...Shadow` and `...SoftImplements`: one
  handler per kind of annotated member [AnnotatedMixinElementHandler.java:AnnotatedMixinElementHandler].
- `Mappings`: implements `IMappingConsumer`. It contains `UniqueMappings` and
  the public static `MappingConflictException` [Mappings.java:Mappings].

### Subpackages

- [ext](./ext/EGM.md): `SpecialPackages`, the package prefixes (`java/`,
  `javax/`, `sun/`, `com/sun/`) that produce no "no mapping" warnings.
- [fg3](./fg3/EGM.md): the ForgeGradle 3 TSRG service. It registers `searge`
  when `mappingTypes` contains `tsrg`.
- [interfaces](./interfaces/EGM.md): the AP's internal contracts, the
  `MessageType` severity table and `CompilerEnvironment` detection.
- [mapping](./mapping/EGM.md): mapping provider, consumer and writer
  interfaces, with SRG and TSRG implementations.
- [mcp](./mcp/EGM.md): the MCP SRG service. It provides `searge`, unless
  `tsrg` is requested, and `notch`.
- [mirror](./mirror/EGM.md): type, member and annotation handles, and the
  `TypeUtils` descriptor helpers.
- [service](./service/EGM.md): `ServiceLoader` discovery of
  `IObfuscationService`s and `ObfuscationTypeDescriptor`.
- [struct](./struct/EGM.md): `Message` and `InjectorRemap`, which delays
  injector errors.
- [validation](./validation/EGM.md): `ParentValidator` (EARLY pass) and
  `TargetValidator` (LATE pass).

## Invariants

### Processing lifecycle

- `AnnotatedMixins` is created once for each `ProcessingEnvironment` and kept
  in a static map. Both processors share it. The constructor runs these steps
  in order [AnnotatedMixins.java:AnnotatedMixins]:
  1. Detect the `CompilerEnvironment`.
  2. Call `MessageType.applyOptions`.
  3. Pass the `Messager` to `MessageRouter.setMessager`.
  4. Check `pluginVersion`.
  5. Print the INFO banner.
  6. Create the `TargetMap`.
  7. Create the `ObfuscationManager` and call `init()`.
  8. Create the validators `ParentValidator` and `TargetValidator`.
  9. Fill the token cache.
- On a round that is not the last, each processor runs `processMixins` and
  then its own element scans, then `postProcess`. On the round where
  `processingOver()` is true, it runs only `postProcess`. `process` always
  returns `true`, so these processors claim the annotations
  [MixinObfuscationProcessorTargets.java:process].
- `processMixins` first clears the list of mixins for the current pass. It
  then registers each `@Mixin` element whose kind is `CLASS` or `INTERFACE`.
  Registration is skipped for a qualified name that is already registered. A
  new mixin's targets go into the `TargetMap`, and the EARLY validators run
  on it [AnnotatedMixins.java:registerMixin].
- `postProcess` calls `onPassCompleted` [AnnotatedMixins.java:onPassCompleted]:
  1. It writes the `TargetMap` unless `disableTargetExport` equals `true`.
  2. On a normal round, it runs LATE validation on the mixins registered
     during this processor call.
  3. On the last round, it runs FINAL validation on every mixin. The first
     FINAL pass also runs a merge check on every mixin method that was not
     registered as `@Overwrite`, `@Shadow`, an injector, `@Accessor` or
     `@Invoker`. When a non-imaginary target has a matching method, the check
     raises `METHOD_VISIBILITY` for a visibility mismatch. A missing match
     raises nothing [AnnotatedMixin.java:runFinalValidation],
     [AnnotatedMixinElementHandlerOverwrite.java:registerMerge].
- After `onPassCompleted`, `MixinObfuscationProcessorInjection` calls
  `writeReferences()`. `MixinObfuscationProcessorTargets` calls
  `writeReferences()` and then `writeMappings()`. This happens on every round,
  including the last one. Exceptions only print a stack trace
  [MixinObfuscationProcessorTargets.java:postProcess].
- An injector annotation on an element whose parent is not a `TypeElement`
  throws `IllegalStateException("@<Name> element has unexpected parent with type <type>")`
  [MixinObfuscationProcessorInjection.java:processInjectors].

### Remap flags and targets

- A mixin is remapped only when `@Mixin(remap)` is not explicitly set to
  false and the mixin has at least one target [AnnotatedMixin.java:AnnotatedMixin].
- For each member annotation, an explicit `remap` value is used. Without one,
  the mixin's own remap flag applies. For `@At`, an explicit `@At.remap` wins,
  and otherwise the injector's flag applies
  [AnnotatedMixins.java:shouldRemap],
  [AnnotatedMixinElementHandlerInjector.java:AnnotatedElementInjectionPoint].
- Classes listed in `@Mixin.value` become public targets. Strings in
  `@Mixin.targets` become private targets, and each needs a type handle. A
  `@Pseudo` mixin gets simulated handles for its string targets. Duplicate
  targets are skipped [AnnotatedMixin.java:initTargets].
- If the current kind of `MIXIN_SOFT_TARGET_NOT_FOUND`, `..._NOT_RESOLVED` or
  `..._IS_PUBLIC` is `ERROR` when that message fires, `initTargets` returns at
  once. `NOT_FOUND` is `ERROR` by default. Later string targets are then not
  processed, and `MIXIN_NO_TARGETS` is not raised. When `NOT_FOUND` is not an
  error, the target that was not found falls back to a simulated handle
  [AnnotatedMixin.java:initTargets].
- `AnnotatedMixins.getTypeHandle(name)` first replaces `/` with `.`. It then
  tries these in order and uses the first that succeeds
  [AnnotatedMixins.java:getTypeHandle]:
  1. `TypeHandleASM.of`, which reads classpath bytes, when the package
     resolves.
  2. A mirror `TypeElement`. Names containing `$` are retried with `.`, first
     by replacing every `$`, then by trying combinations when there are 2 to
     10 `$` characters.
  3. An imaginary `TypeHandle(pkg, name)`.
  4. `null`, when even the package cannot be resolved.

### Options

`getOption(key)` reads processor options first. If the key is absent, it falls
back to `mixin.properties`, loaded once from the root of
`StandardLocation.SOURCE_PATH`. Errors while loading that file are ignored
[AnnotatedMixins.java:getProperties].

`getOptions(key)` splits on `,` without trimming. `getOption(key, boolean)`
uses `Boolean.parseBoolean`. Most flags are compared with
`"true".equalsIgnoreCase(...)` [AnnotatedMixins.java:getOptions].

| Key (constant in `SupportedOptions`) | Effect | Source |
| --- | --- | --- |
| `tokens` (`TOKENS`) | Constraint token values. Whitespace is removed and the value is uppercased, then split on `;` or `,`. Each entry must match `^([A-Z0-9\-_\.]+)=([0-9]+)$`; others are ignored. Tokens that are not cached fall back to an option with the same name, parsed as an `int` | [AnnotatedMixins.java:initTokenCache], [AnnotatedMixins.java:getToken] |
| `outRefMapFile` (`OUT_REFMAP_FILE`) | Name or path of the refmap file. When unset, no refmap is written | [ReferenceManager.java:write] |
| `disableTargetValidator` | `true` skips `TargetValidator` | [TargetValidator.java:validate] |
| `disableTargetExport` | `true` skips writing the `TargetMap` session file | [AnnotatedMixins.java:onPassCompleted] |
| `disableOverwriteChecker` | `true` skips the `@Overwrite` javadoc checks | [AnnotatedMixinElementHandlerOverwrite.java:registerOverwrite] |
| `overwriteErrorLevel` | `error` raises `OVERWRITE_DOCS` to `ERROR` | [IMessagerEx.java:applyOptions] |
| `defaultObfuscationEnv` | Lowercased key of the default type, `searge` if unset. That type's mappings are also written to the refmap root `mappings` | [ObfuscationType.java:isDefault], [ReferenceManager.java:addMapping] |
| `dependencyTargetsFile` | Tab-separated `<mixin>\t<target>` lines imported into the `TargetMap`. A path that is not a file is ignored | [TargetMap.java:readImports] |
| `mappingTypes` | Comma-separated list. The exact element `tsrg` makes the FG3 service provide `searge` instead of MCP | `fg3/` and `mcp/` |
| `pluginVersion` | MixinGradle version, added to the banner. A version below `0.7` gives a warning | [AnnotatedMixins.java:checkPluginVersion] |
| `quiet` | `true` disables `INFO` messages | [IMessagerEx.java:applyOptions] |
| `showMessageTypes` | `true` decorates messages as `[TYPE] msg` | [IMessagerEx.java:applyOptions] |
| `disableInterfaceMixins` | `true` enables the `INJECTOR_IN_INTERFACE` error | [IMessagerEx.java:applyOptions] |

`getAllOptions()` also adds every service option and every `MSG_<MessageType>`
key. The service options are the MCP, FG3 and example keys listed in the
`mcp`, `fg3` and example JSON contracts [SupportedOptions.java:getAllOptions].

For each obfuscation type, the extra-input option is split on `;` and each
part is trimmed. A type is supported, and gets an environment, when at least
one input-file option is set, whether or not the file exists
[ObfuscationType.java:getInputFileNames], [ObfuscationType.java:isSupported].

### Refmap output

- `ReferenceManager.write()` does nothing if `outRefMapFile` is unset.
  Otherwise it serializes the in-memory `ReferenceMapper`
  (`src/main/java/org/spongepowered/asm/mixin/refmap/ReferenceMapper.java`)
  as pretty-printed JSON with two top-level maps
  [ReferenceManager.java:write]:
  - `mappings`: entries for the default type only.
  - `data`: entries keyed by obfuscation type key.
  - Inside each map, keys are the mixin's internal class name, then the
    reference string as written in source. `ReferenceMapper` removes
    whitespace from the reference and drops identity mappings.
- A name containing `/`, `\` or `:` is opened as a filesystem path, and parent
  directories are created. Any other name is created with
  `Filer.createResource(CLASS_OUTPUT, "", name)`. If that throws, the call is
  retried with only the file's base name [ReferenceManager.java:newWriter].
- On success the AP emits an INFO message. For a filesystem path it is
  `Writing refmap to <absolute path>`. For a `Filer` resource it is
  `Writing refmap to (<uri scheme>) <path>` [ReferenceManager.java:newWriter].
- If opening the file fails, the AP emits `MessageType.ERROR` with the text
  `Cannot write refmap to (<name>): <ExceptionClass>: <message>`, and nothing
  is written [ReferenceManager.java:newWriter].
- `write()` runs on every round from both processors, and each call opens the
  output again [MixinObfuscationProcessorInjection.java:postProcess].
- Adding a different value for an existing `(type, mixin, reference)` throws
  `ReferenceConflictException` unless conflicts are allowed. Mappings for the
  default type are written under both `data[<key>]` and `mappings`
  [ReferenceManager.java:addMapping].
- Field and method references are rewritten with `MemberInfo`. Field
  descriptors are remapped through the environment
  [ReferenceManager.java:addFieldMapping].

### Generated mapping files

- Each mixin gets one `Mappings` consumer from `createMappingConsumer()`.
  Handlers write to it through `Mappings.asUnique()`. Adding the same `from`
  with a different `to` for one type throws `MappingConflictException`.
  Adding the same pair again does nothing [Mappings.java:checkForExistingMapping].
- Generated entries are owned by the mixin class. Both `from` and `to` use the
  mixin's `classRef` as owner, mapping the name used in the mixin to the
  obfuscated name [AnnotatedMixinElementHandler.java:addMethodMapping].
- `writeMappings()` collects every consumer's sets for each environment.
  It passes them to that environment's writer with the type's output-file
  option. A `null` output name means no file is written
  [ObfuscationEnvironment.java:writeMappings].

### Obfuscation lookups

- Mapping files load lazily on the first lookup [ObfuscationEnvironment.java:initMappings]:
  - For each input that `isFile()`, the AP emits INFO
    `Loading <type> mappings from <abs path>` and reads the file. Read
    exceptions only print a stack trace.
  - If no file loads, the AP emits `Kind.ERROR`
    `No valid input files for <type> could be read, processing may not be sucessful.`
    and clears the provider. "sucessful" is misspelled in the source.
- `getObfMethod(MappingMethod)` follows `getSuper()` until the provider returns
  a mapping. It then remaps the owner to its obfuscated class
  [ObfuscationEnvironment.java:getObfMethod].
- `getObfClass(className)` returns `ObfuscationData` whose default value is the
  unmapped name [ObfuscationDataProvider.java:getObfClass].
- When no environment maps a method name, constructor lookups remap only the
  owner and descriptor [ObfuscationDataProvider.java:getRemappedMethod].
- Recursive field and method lookups check each interface, then the
  superclass, at every level of the hierarchy. A found entry is moved back to
  the obfuscated owner of the original class [ObfuscationDataProvider.java:getObfEntryRecursive].
- `ObfuscationType.create` throws `IllegalArgumentException("Obfuscation type with key <k> was already registered")`.
  `get` throws `IllegalArgumentException("Obfuscation type with key <k> was not registered")`.
  `createEnvironment` wraps reflection failures in `RuntimeException`
  [ObfuscationType.java:create].

### Member handling rules

- `@Shadow`: the prefix comes from `prefix`, defaulting to `shadow$`, and is
  removed before the lookup. Target checks also try `aliases` and the name
  without the prefix [AnnotatedMixinElementHandler.java:ShadowElementName].
- `@Overwrite`: the method is checked against every target, including its
  visibility. When remapping fails for any target, the javadoc checks are
  skipped. Otherwise the javadoc is checked, ignoring case, for a comment, an
  `@author` tag and an `@reason` tag [AnnotatedMixinElementHandlerOverwrite.java:registerOverwrite].
- Injectors:
  - Each `method` string and each `target` `@Desc` is parsed with
    `TargetSelector.parse`.
  - A target without a descriptor uses the first method in the target with a
    matching name.
  - Owners are removed from refmap entries when the selector has no owner and
    the mixin has several targets, and for simulated targets.
  - A `<clinit>` target with no mapping is skipped without a message.
  [AnnotatedMixinElementHandlerInjector.java:registerInjector]
- `@At` and `@Slice`:
  - `@At.target` must be fully qualified (owner and descriptor) for remapping.
  - For `NEW`, the value of the `class` arg in `args` is also remapped.
  - The message subject format is `@At(<TYPE>.<target>)`,
    `@Slice[<id>](from=@At(...))` or `@Slice[<id>](to=@At(...))`.
  [AnnotatedMixinElementHandlerInjector.java:registerInjectionPoint], [AnnotatedMixin.java:registerInjector]
- `@Accessor` and `@Invoker`:
  - The target name comes from `value`, or is inflected from the method name
    when `value` is empty.
  - A void return makes an accessor a setter; any other return makes it a
    getter.
  - An `@Invoker` is a factory when it returns a declared type and one of
    these holds: `value` is `<init>`, `value` names the target, or (with no
    `value`) its name uses a factory prefix and names `<init>` or the target's
    simple name.
  - Refmap entries have their owners removed.
  [AnnotatedMixinElementHandlerAccessor.java:registerAccessor]
- `@Implements`: does nothing when the mixin is not remapped. Entries with
  `Remap.NONE` are skipped. With `ONLY_PREFIXED`, only prefixed methods are
  mapped. Super-interfaces are walked recursively
  [AnnotatedMixinElementHandlerSoftImplements.java:process].
- `@SuppressWarnings` suppresses only messages whose final kind is
  `Kind.WARNING`. The token is searched on the element and then on each
  enclosing element [AnnotatedMixins.java:shouldSuppress].

| `SuppressedBy` token | Suppresses |
| --- | --- |
| `constraints` | `INVALID_CONSTRAINT` |
| `visibility` | `METHOD_VISIBILITY` |
| `target` | `TARGET_ELEMENT_NOT_FOUND` |
| `mapping` | `NO_OBFDATA_FOR_CLASS`, `NO_OBFDATA_FOR_FIELD`, `NO_OBFDATA_FOR_METHOD` |
| `overwrite` | `OVERWRITE_DOCS` |
| `default-package` | `MIXIN_SOFT_TARGET_IS_PUBLIC`, when the target is in the unnamed package |
| `public-target` | `MIXIN_SOFT_TARGET_IS_PUBLIC`, other cases |
| `unresolvable-target` | `MIXIN_SOFT_TARGET_NOT_RESOLVED` |
| `rawtypes` | Declared only. No call site in this package |

[SuppressedBy.java:SuppressedBy]

### Diagnostics emitted from this package

Each entry gives the `MessageType` (default kind) and the message text, with
`<...>` marking interpolated values. The service and validator messages are
listed in `service/` and `validation/`.

- Processor scans [MixinObfuscationProcessorTargets.java:process], [MixinObfuscationProcessor.java:processMixins]:
  - `MIXIN_ON_INVALID_TYPE` (ERROR): `Found an @Mixin annotation on an element which is not a class or interface`
  - `INJECTOR_ON_NON_METHOD_ELEMENT` (WARNING, no element position): `Found an @<Name> annotation on an element which is not a method: <elem>`
  - `ERROR`: `Unexpected parent with type <type>`
  - `SHADOW_ON_INVALID_ELEMENT`: `Element is not a method or field`
  - `OVERWRITE_ON_NON_METHOD_ELEMENT` and `ACCESSOR_ON_NON_METHOD_ELEMENT`: `Element is not a method`
  - `SOFT_IMPLEMENTS_ON_INVALID_TYPE`: `Found an @Implements annotation on an element which is not a class or interface`
- Registration on a class that is not a mixin [AnnotatedMixins.java:registerAccessor]:
  - `ACCESSOR_ON_NON_MIXIN_METHOD`: `Found @Accessor annotation on a non-mixin method` or `Found @Invoker annotation on a non-mixin method`
  - `OVERWRITE_ON_NON_MIXIN_METHOD`: `Found @Overwrite annotation on a non-mixin method`
  - `SHADOW_ON_NON_MIXIN_ELEMENT`: `Found @Shadow annotation on a non-mixin field` or `... non-mixin method`
  - `INJECTOR_ON_NON_MIXIN_METHOD`: `Found @<Name> annotation on a non-mixin method`
  - `SOFT_IMPLEMENTS_ON_NON_MIXIN`: `Found @Implements annotation on a non-mixin class`
- Start-up [AnnotatedMixins.java:AnnotatedMixins]:
  - `INFO`: `SpongePowered MIXIN Annotation Processor Version=<ver>`, followed by ` (MixinGradle Version=<v>)` when `pluginVersion` is set
  - `Kind.WARNING`: `MixinGradle version <v> is out of date. Update to the recommended version <r>`, where `<r>` is the `VersionNumber` parsed from `RECOMMENDED_MIXINGRADLE_VERSION = "0.7"`
  - `Kind.WARNING`: `Could not read from specified imports file: <name>`
- Targets, positioned on the mixin and its `@Mixin` [AnnotatedMixin.java:initTargets]:
  - `MIXIN_SOFT_TARGET_NOT_FOUND` (ERROR): `Mixin target <name> could not be found`
  - `MIXIN_SOFT_TARGET_NOT_RESOLVED` (WARNING): `Mixin target <name> could not be fully resolved.`
  - `MIXIN_SOFT_TARGET_IS_PUBLIC` (WARNING): `Mixin target <name> is public and should be specified in value`. The text says `must` when the kind is ERROR
  - `MIXIN_NO_TARGETS` (ERROR): `Mixin has no targets`
  - `WARNING`: `Error processing public targets: <Ex>: <msg>` or `Error processing private targets: ...`
- Shared checks [AnnotatedMixinElementHandler.java:validateTargetMethod]:
  - `TARGET_ELEMENT_NOT_FOUND` (WARNING): `Cannot find target for <type> method in <target>`, `Cannot find target for <type> field in <target>` or `Cannot find target method "<name><desc>" for <subject> in <target>`
  - `METHOD_VISIBILITY` (WARNING): `<VIS> <type> method cannot reduce visibility of <VIS> method in <target>` or `<VIS> <type> method will upgrade visibility of PRIVATE method in <target>`
  - `CONSTRAINT_VIOLATION` (ERROR) and `INVALID_CONSTRAINT` (WARNING): the exception message
- Accessors [AnnotatedMixinElementHandlerAccessor.java:registerAccessorForTarget]:
  - `ACCESSOR_NAME_UNRESOLVED`: `Cannot inflect accessor target name`
  - `ACCESSOR_TYPE_UNSUPPORTED`: `Unsupported accessor type`
  - `ACCESSOR_ATTACH_ERROR`: the exception message
  - `ACCESSOR_TARGET_NOT_FOUND`: `Could not locate @Accessor target <name> in target <target>`, or the same with `@Invoker`
  - `NO_OBFDATA_FOR_ACCESSOR`: `Unable to locate obfuscation mapping[ in target <t>] for @Accessor target <name>`. This text is also used for invokers
  - `ACCESSOR_MAPPING_CONFLICT`: `Mapping conflict for @Accessor target ...` or `Mapping conflict for @Invoker target ...`
  - `FACTORY_INVOKER_RETURN_TYPE`: `Invalid Factory @Invoker return type, expected <target> but found <type>`
  - `FACTORY_INVOKER_NONSTATIC`: `Factory @Invoker must be static`
- Injectors [AnnotatedMixinElementHandlerInjector.java:registerInjector]:
  - `INJECTOR_IN_INTERFACE`: `Injector in interface is unsupported`. This type is disabled unless `disableInterfaceMixins` is set
  - `TARGET_SELECTOR_VALIDATION`: the `InvalidSelectorException` message
  - `MISSING_INJECTOR_DESC_SIMULATED` (OTHER): `<@Ann> target '<ref>' in @Pseudo mixin will not be obfuscated`
  - `MISSING_INJECTOR_DESC_MULTITARGET` (ERROR) or `..._SINGLETARGET` (WARNING): `<@Ann> target requires method signature because enclosing type information for <target> is unavailable` or `Unable to determine descriptor for <@Ann> target method`
  - `NO_OBFDATA_FOR_TARGET` (ERROR) or `NO_OBFDATA_FOR_CTOR` (WARNING): `Unable to locate obfuscation mapping for <@Ann> target <name>`. This message is held in `InjectorRemap` and dropped if any `@At` in the same injector is remapped
  - `BARE_REFERENCE` (WARNING): `Coerced <Multi-target|Target> reference has conflicting descriptors for <name>: Storing bare references [<values>] in refMap`
  - `INJECTOR_MAPPING_CONFLICT`: `<Multi-target|Target> reference conflict for <name>: <ref> -> <new> previously defined as <old>` or `Unexpected reference conflict for <subject>: ...`
  - `INJECTOR_TARGET_NOT_FULLY_QUALIFIED`: `<subject> is not fully qualified, missing <owner and descriptor|owner|descriptor>`
  - `NO_OBFDATA_FOR_CLASS`: `Unable to locate class mapping for <subject> '<target>'`
  - `NO_OBFDATA_FOR_FIELD`: `Unable to locate field mapping for <subject> '<ref>'`
  - `NO_OBFDATA_FOR_METHOD`: `Unable to locate method mapping for <subject> '<ref>'`
  - The last three are skipped for `SpecialPackages` owners.
- Overwrites [AnnotatedMixinElementHandlerOverwrite.java:registerOverwriteForTarget]:
  - `NO_OBFDATA_FOR_OVERWRITE` (ERROR), or `NO_OBFDATA_FOR_STATIC_OVERWRITE` (WARNING) when the element's reflective `isStatic()` returns true: `Unable to locate obfuscation mapping for @Overwrite method`
  - `OVERWRITE_MAPPING_CONFLICT`: `Mapping conflict for @Overwrite method: <new> for target <t> conflicts with existing mapping <old>`
  - `OVERWRITE_DOCS`: `@Overwrite is missing javadoc comment`, `@Overwrite is missing an @author tag` or `@Overwrite is missing an @reason tag`
- Shadows [AnnotatedMixinElementHandlerShadow.java:registerShadowForTarget]:
  - `NO_OBFDATA_FOR_SHADOW`, or `NO_OBFDATA_FOR_SIMULATED_SHADOW` for a simulated target: `Unable to locate obfuscation mapping[ in target <t>] for @Shadow <field|method>`
  - `SHADOW_MAPPING_CONFLICT`: `Mapping conflict for @Shadow <field|method>: ...`
- Soft implements [AnnotatedMixinElementHandlerSoftImplements.java:process]:
  - `SOFT_IMPLEMENTS_EMPTY`: `Empty @Implements annotation`
  - `ERROR`: `Unexpected error: <Ex>: <msg>`
  - `NO_OBFDATA_FOR_SOFT_IMPLEMENTS`: `No obfuscation mapping for soft-implementing method`, only when `Remap.forceRemap()` is true
- IDE with no obfuscation data [ObfuscationManager.java:init]:
  - `NOTE`: `No obfuscation data are available and an IDE (<name>) was detected, quenching error levels `
  - The message prefix becomes `(Mixin AP) `.
  - Every `NO_OBFDATA_*` type is lowered to `NOTE`, and `PARENT_VALIDATOR` and
    `TARGET_VALIDATOR` are lowered to `WARNING`.

### State that lasts as long as the class loader

- The `ObfuscationType` registry is a static map. `ObfuscationServices` fills
  it only once (see `service/`). Each type keeps the
  `IMixinAnnotationProcessor` it was created with, and environments read
  their input and output options through that processor. As a result, later
  compilations in the same class loader reuse the types, and the option
  values, of the first compilation [ObfuscationType.java:types], [ObfuscationEnvironment.java:ObfuscationEnvironment].
- `AnnotatedMixins.instances` is a static map keyed by `ProcessingEnvironment`.
  Entries are never removed [AnnotatedMixins.java:instances].

## Side Effects

- Compiler diagnostics go through `processingEnv.getMessager()`. Messages
  whose `MessageType` is disabled are dropped, except the delayed injector
  messages described in `struct/`. Warnings can be suppressed with
  `@SuppressWarnings` tokens [AnnotatedMixins.java:printMessage].
- The refmap file is written to the path, or to `CLASS_OUTPUT`, named by
  `outRefMapFile` [ReferenceManager.java:write].
- Mapping files are written for each obfuscation type to the path named by
  that type's output option (see `mapping/`) [ObfuscationManager.java:writeMappings].
- Input mapping files and `dependencyTargetsFile` are read from disk.
  `mixin.properties` is read from the source path [ObfuscationEnvironment.java:initMappings], [TargetMap.java:readImports].
- `TargetMap` session file:
  - It reads the system property `mixin.target.mapid` and then sets it to the
    session id. A new session id is `System.currentTimeMillis()`.
  - Each pass, Java serialization writes the map in append mode to
    `${java.io.tmpdir}/mixin-targetdb-<sessionId>.tmp`, which is marked
    `deleteOnExit`.
  - `create(sessionId)` reads one object back from an existing session file.
  [AnnotatedMixins.java:initTargetMap], [TargetMap.java:write]
- `MessageRouter.setMessager(...)` (in `src/main`) receives the compiler
  `Messager` [AnnotatedMixins.java:AnnotatedMixins].
- Many I/O failures only print a stack trace and raise no diagnostic: writing
  refmaps or mappings, writing or reading the target map, and reading mapping
  inputs [ReferenceManager.java:write].

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileApJava`
- Run: `mise exec -- ./gradlew test`. Note that `src/test` does not exist yet,
  so no tests cover this package today.
