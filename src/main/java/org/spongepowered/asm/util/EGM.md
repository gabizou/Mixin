# org.spongepowered.asm.util

## Overview

Package `org.spongepowered.asm.util` holds the utility classes that the whole Mixin pipeline uses (package-info.java). It contains:

- bytecode and descriptor helpers (`Bytecode`, `Handles`, `ObfuscationUtil`, `SignaturePrinter`)
- annotation-node accessors (`Annotations`)
- generic class-signature parsing and merging (`ClassSignature`)
- local-variable-table reconstruction (`Locals`)
- small parsers (`ConstraintParser`, `Quantifier`, `VersionNumber`, `JavaVersion`)
- a boxed text formatter (`PrettyPrinter`)
- general-purpose holders (`Constants`, `Counter`, `ReEntranceLock`, `Files`, `LanguageFeatures`)

The main source set compiles at Java 1.6 against ASM 9.5, Guava 21 and Gson 2.2.4 (`build.gradle`, `gradle.properties`). Most classes here are final or abstract utility classes with private constructors and static methods. Unless a section notes a dependency on `MixinService`, `ClassInfo`, or the console, they can be unit-tested with ASM tree nodes built in memory.

Behaviour marked *(observed)* was confirmed by running the compiled classes (JDK 16, ASM 9.5) in `jshell`.

## Interface

### Subpackages

- `asm`: [./asm/EGM.md](./asm/EGM.md) covers ASM version detection (`ASM`), the nest-attribute adapter (`ClassNodeAdapter`), `IAnnotationHandle`, `MarkerNode`, `MethodNodeEx`, `MethodVisitorEx` and `MixinVerifier`.
- `logging`: [./logging/EGM.md](./logging/EGM.md) covers `MessageRouter`, a static switch between the annotation processor's `Messager` and the `"mixin"` logger.
- `perf`: [./perf/EGM.md](./perf/EGM.md) covers `Profiler`, which has a global registry and an `active` flag, section stacks and phases.
- `throwables`: [./throwables/EGM.md](./throwables/EGM.md) covers `ConstraintViolationException`, `InvalidConstraintException`, `LVTGeneratorError` and `SyntheticBridgeException`.

### `Bytecode` (final, static helpers)

#### Visibility

- `enum Visibility` has the ordinal order `PRIVATE` < `PROTECTED` < `PACKAGE` < `PUBLIC`, so `PROTECTED` ranks below `PACKAGE`. `isAtLeast(null)` returns true and `isLessThan(null)` returns false. `MASK = ACC_PRIVATE|ACC_PROTECTED|ACC_PUBLIC` [Bytecode.java:Visibility].
- `getVisibility(MethodNode|FieldNode)` tests `access & 0x7` in this order: protected, private, public, then `PACKAGE`. Conflicting bits such as public plus protected therefore yield `PROTECTED` [Bytecode.java:getVisibility].
- `setVisibility(node, Visibility|int)` replaces only the `MASK` bits of `access` and keeps all other flags [Bytecode.java:setVisibility].
- `isVirtual(MethodNode)` returns false for `null`, for static methods and for private methods. It returns true for protected, package and public instance methods [Bytecode.java:isVirtual].
- `hasFlag(node, flag)` returns `(access & flag) == flag`, so `flag == 0` is always true and multi-bit flags require every bit. `compareFlags(a, b, flag)` returns true when both nodes have the flag or both lack it [Bytecode.java:hasFlag]. `isStatic(MethodNode|FieldNode)` [Bytecode.java:isStatic].

#### Descriptors and arguments

- `getArgsSize(Type[])` sums `Type.getSize()`, so `long` and `double` count as 2. `getArgsSize(args, start, end)` covers indices from `start` up to `min(end, args.length)`, exclusive [Bytecode.java:getArgsSize].
- `getFirstNonArgLocalIndex(MethodNode)` returns `getArgsSize(args) + 1`, without the `+ 1` for static methods. `getFirstNonArgLocalIndex(Type[], boolean includeThis)` [Bytecode.java:getFirstNonArgLocalIndex].
- `loadArgs(args, insns, start[, end[, casts]])` appends an `xLOAD` for each argument, starting at local `start` and advancing by the argument's size [Bytecode.java:loadArgs]:
  - When `casts[i]` is non-null, it appends a `CHECKCAST` after that load.
  - `end` is an exclusive local-slot bound, checked after each load and only when `end >= start`. As a result `loadArgs(args, l, 0, 0)` emits one load, not zero *(observed)*.
  - The three-argument form passes `end = -1` and loads every argument. For example, `(I, J, I)` from slot 1 loads slots 1, 2 and 4 *(observed)*.
- `generateDescriptor(Object ret, Object... args)` converts each part as follows [Bytecode.java:generateDescriptor]:
  - A `String` is used verbatim, a `Type` contributes its descriptor, a `Class` goes through `Type.getDescriptor`, and `null` contributes `""`.
  - A `null` return type becomes `V`. For example, `(null, "I", LONG_TYPE, String.class, null)` gives `(IJLjava/lang/String;)V` *(observed)*.
- `getDescriptor(Type... args)` returns only `"(" + args + ")"`, with no return type. `getDescriptor(Type ret, Type... args)` appends `ret`. The call `getDescriptor(t1, t2)` is a compile-time ambiguity error, so pass a `Type[]` explicitly *(observed)* [Bytecode.java:getDescriptor].
- `changeDescriptorReturnType(desc, ret)` returns `null` when `desc` is `null`, does not start with `(`, or has no `)` at index 1 or later. A `null` `ret` returns `desc` unchanged. Otherwise it replaces everything after the last `)` [Bytecode.java:changeDescriptorReturnType].
- `getSimpleName(String)` returns the text after the last `/` with every `;` removed. With no `/` present, it drops the first character: `"Ljava/lang/String;"` gives `String`, `"LFoo;"` gives `Foo`, and `"Foo"` gives `oo` *(observed)* [Bytecode.java:getSimpleName].
- `getSimpleName(Type)` returns the descriptor for primitives and `void`. For object and array types it applies `getSimpleName(String)` to the dotted class name, which drops the first character: `String` gives `ava.lang.String` *(observed)* [Bytecode.java:getSimpleName].
- `getTypes(Class...)` returns a `Type[]` [Bytecode.java:getTypes].
- `getBoxingType(Type)` and `getUnboxingMethod(Type)` are indexed by `getSort()`. Booleans through doubles map to their wrapper (`java/lang/Integer`, ...) and `xxxValue` method. `VOID`, `ARRAY`, `OBJECT`, `METHOD` and `null` map to `null` [Bytecode.java:getBoxingType].

#### Constants

- `CONSTANTS_INT` holds `ICONST_M1` to `ICONST_5`. Also public: `CONSTANTS_FLOAT`, `CONSTANTS_DOUBLE`, `CONSTANTS_LONG`, and `CONSTANTS_ALL`, which holds `ACONST_NULL`, the `xCONST_*` opcodes, `BIPUSH`, `SIPUSH`, `LDC`, `CHECKCAST` and `INSTANCEOF`. These arrays are public and mutable [Bytecode.java:CONSTANTS_INT].
- `isConstant(insn)`: `null` returns false. Otherwise it returns true when the opcode is in `CONSTANTS_ALL`, which includes `CHECKCAST` and `INSTANCEOF` [Bytecode.java:isConstant].
- `getConstant(insn)` [Bytecode.java:getConstant]:
  - `null` returns `null`, and an LDC returns its `cst`.
  - `BIPUSH` and `SIPUSH` return an `Integer`. Any other `IntInsnNode`, such as `NEWARRAY`, throws `IllegalArgumentException("IntInsnNode with invalid opcode N in getConstant")`.
  - `NEW` and `ANEWARRAY` return `null`. `CHECKCAST` and `INSTANCEOF` return `Type.getObjectType(desc)`.
  - `ACONST_NULL` returns `Type.VOID_TYPE`, not `null`, and `xCONST_*` returns the boxed value. Any other opcode returns `null`.
- `getConstantType(insn)` [Bytecode.java:getConstantType]:
  - An LDC `Integer`, `Float`, `Long`, `Double`, `String` or `Type` maps to `I`, `F`, `J`, `D`, `Ljava/lang/String;` or `Ljava/lang/Class;`. Any other payload throws `IllegalArgumentException`.
  - `CHECKCAST` and `INSTANCEOF` map to `Ljava/lang/Class;`, and `ACONST_NULL` maps to `V`.
  - `BIPUSH` and `SIPUSH` map to `I`. `NEWARRAY` maps to `null` without throwing.

#### Opcode names and node descriptions

- `getOpcodeName(int)` [Bytecode.java:getOpcodeName]:
  - It reflects over the `Opcodes` fields, starting at the `UNINITIALIZED_THIS` field, and returns the name of the first `int` field that matches.
  - Opcodes below 1 are not named, so `0` gives `"0"`. Negative values give `"UNKNOWN"`, and unmatched values give the decimal string (`500` gives `"500"`) *(observed)*.
  - `getOpcodeName(AbstractInsnNode null)` returns `""`.
- `parseOpcodeName(String)` [Bytecode.java:parseOpcodeName]:
  - `null` gives -1.
  - Numeric strings from `1` to `199` are parsed as numbers. An `Opcodes.` prefix is stripped.
  - Names must match `^[A-Z][A-Z0-9_]+$`, so lowercase names give -1. Names declared before `UNINITIALIZED_THIS`, such as `ACC_PUBLIC`, also give -1. `"0"` and `"200"` give -1 *(observed)*.
- `describeNode(insn[, listFormat])` returns the node's class simple name minus `Node`, followed by details [Bytecode.java:describeNode]:
  - For example `VarInsn [ALOAD] 0` and `Insn [RETURN] ` (with a trailing space).
  - `null` gives `"   null           "` in list format and `"null"` otherwise. Labels give `[label]`, and a `MarkerNode` gives `[label] Marker type=N`.

#### Lookup, delegate initialisers and bridge methods

- `findMethod(ClassNode, name, desc)` returns an exact match or `null`. `findInsn(MethodNode, opcode)` returns the first matching instruction or `null`. `methodHasLineNumbers(MethodNode)`. `getMaxLineNumber(ClassNode, min, pad)` returns `max(min, maxLine + pad)` [Bytecode.java:findMethod].
- `findDelegateInit(ctor, superName, ownerName)` [Bytecode.java:findDelegateInit]:
  - It returns `DelegateInitialiser.NONE` when the method is not `<init>` or no delegate call exists. `NONE` has `insn == null`, `isPresent == false` and `toString() == "this"`.
  - Each `NEW` cancels one later `INVOKESPECIAL <init>`. The first unmatched `<init>` call whose owner is `superName` or `ownerName` gives `isSuper` true or false respectively.
- `cloneLabels(InsnList)` maps each `LabelNode` to a new `LabelNode` that shares the same underlying `Label` [Bytecode.java:cloneLabels].
- `compareBridgeMethods(a, b)` walks both instruction lists in lockstep and skips positions where `a` holds a label [Bytecode.java:compareBridgeMethods]:
  - It throws `SyntheticBridgeException` with `BAD_INVOKE_NAME` or `BAD_INVOKE_DESC` when invoke name or descriptor differ; owners and opcodes are not compared for method instructions.
  - It throws `BAD_INSN` for a different opcode, `BAD_LOAD` for a different `var`, `BAD_CAST` for a different `CHECKCAST` descriptor, and `BAD_LENGTH` (with null nodes) for unequal lengths.
  - A `MethodInsnNode` in `a` facing another node type in `b` throws `ClassCastException`.

#### Whole-class operations and printing

- `merge(ClassNode source, ClassNode dest)` [Bytecode.java:merge]:
  - A `null` source does nothing. A `null` dest throws `NullPointerException("Target ClassNode for merge must not be null")`.
  - It sets `version = max(...)` and appends the source's interfaces, annotations, type annotations, attributes, inner classes, fields and methods to `dest`. It copies references without removing duplicates.
- `replace(source, dest)` copies name, signature, superName, version, access, source and outer info, and `module` when the ASM API is ASM6 or later. It then clears the `dest` lists and merges [Bytecode.java:replace].
- `textify(ClassNode|MethodNode, OutputStream)` writes ASM trace text [Bytecode.java:textify]. `dumpClass(ClassNode|byte[])` writes with `COMPUTE_MAXS|COMPUTE_FRAMES` and runs `CheckClassAdapter.verify`, printing to `System.out` [Bytecode.java:dumpClass]. `printMethod`, `printMethodWithOpcodeIndices` and `printNode` print to `System.err` [Bytecode.java:printMethod].

### `Annotations` (final, static helpers)

#### Reading annotations and values

- `getVisible` and `getInvisible` take a `FieldNode`, `MethodNode` or `ClassNode` plus an annotation `Class`. They return the first `AnnotationNode` whose `desc` matches, or `null` [Annotations.java:getVisible].
- `get(List, String desc)` returns `null` for a `null` list [Annotations.java:get].
- `getSingleVisible` and `getSingleInvisible(MethodNode, Class...)` return `null` when no class matches and the node when one matches. More than one match throws `IllegalArgumentException("Conflicting annotations found: [descs]")` [Annotations.java:getSingle].
- `getVisibleParameter` and `getInvisibleParameter(method, cls, index)` look at the method-level annotation when `index < 0`. They return `null` when the index is out of range or the array is `null` [Annotations.java:getParameter].
- `getValue(node, key)` scans every element of `node.values` and returns the element after the first one equal to `key` [Annotations.java:getValue]:
  - It does not step in pairs, so a value equal to a key name misdirects the lookup. With values `["name","value","value","x"]`, looking up `value` returns `"value"` *(observed)*.
  - A `null` node or `null` values gives `null`. `getValue(node)` reads `"value"`, and `getValue(node, key, default)` returns `default` when the result is `null`.
- `getValue(node, key, Class<?> annotationClass)` falls back to the annotation method's declared default through reflection. A `null` class throws `NullPointerException` [Annotations.java:getValue].
- `getValue(node, key, Class<E> enumClass, E default)` [Annotations.java:toEnumValue]:
  - It reads a stored `String[]{desc, name}`.
  - It throws `IllegalArgumentException("The supplied enum class does not match the stored enum value")` when the descriptor's class differs, and `Enum.valueOf` throws on an unknown name.
- `getValue(node, key, boolean notNull)` returns a list; the `notNull` flag is ignored [Annotations.java:getValue]:
  - A stored `List` is returned as the node's own live list.
  - A scalar is wrapped in a new `ArrayList`.
  - A missing key gives the immutable `Collections.emptyList()`.
- `getValue(node, key, boolean, Class<E>)` converts a stored list of enum `String[]` values to enum constants in place, so the annotation node is mutated. A second call on the same node throws `ClassCastException` *(observed)* [Annotations.java:getValue].
- `getDesc(Class)` returns the internal name. `getSimpleName(Class)` and `getSimpleName(AnnotationNode)` return the simple name, the latter through `Bytecode.getSimpleName(desc)` [Annotations.java:getSimpleName].

#### Writing and merging annotations

- `setValue(node, key, value)` [Annotations.java:setValue]:
  - A `null` node does nothing.
  - It replaces the value of an existing key in place, or appends a new key and value.
  - Enums are stored as `String[]{Type.getDescriptor(enumClass), value.toString()}`. A `null` value throws `NullPointerException`.
- `setVisible` and `setInvisible(FieldNode|MethodNode, Class, Object... keyValues)` remove any existing annotation with the same `desc` and append a new node. A non-`String` key throws `IllegalArgumentException("Annotation keys must be strings, ...")`, and a trailing odd value is ignored [Annotations.java:createNode].
- `merge(ClassNode|MethodNode|FieldNode from, to)` copies mergeable annotations into `to`. Each copy replaces the first existing annotation with the same `desc` and shares the node objects [Annotations.java:merge]:
  - Annotations outside `org/spongepowered/asm/mixin` are always mergeable.
  - Inside that package, only `@Overwrite`, `@Intrinsic`, `@Final` and `@Debug` are mergeable.
  - Exceptions are caught and logged as a warning through `MixinService` [Annotations.java:isMergeableAnnotation].

#### Handles

- `handleOf(Object)`:
  - An `IAnnotationHandle` is returned as is, and an `AnnotationNode` is wrapped in a new `Handle`.
  - `null` gives `null`. Anything else throws `IllegalArgumentException("Unsupported annotation type: <class>")` [Annotations.java:handleOf].
- `Annotations.Handle implements IAnnotationHandle` [Annotations.java:Handle]:
  - The constructor is package-private and rejects `null` with `Preconditions.checkNotNull`. `exists()` is always true, and `getNode()` returns the wrapped node.
  - `getDesc()` returns the internal name (`foo/Bar`), not the descriptor *(observed)*. `getTypeValue(key)` defaults to `Type.VOID_TYPE`.
  - `getList(key)` follows `getValue(node, key, false)`, and `getAnnotation(key)` returns `null` when the key is missing. `toString()` returns `"@" + simpleName`.

### `ClassSignature`

- `of(String)` parses with ASM's `SignatureReader`. A `null` signature gives an empty signature whose superclass is `java/lang/Object`. A malformed signature is not thrown: the stack trace goes to `System.err` and the partially built state is returned (`"garbage"` gives `Ljava/lang/Object;`) *(observed)* [ClassSignature.java:read].
- `of(ClassNode)` parses `classNode.signature`. When that is `null`, it generates a signature from `superName` (falling back to `java/lang/Object` when `null`) and `interfaces` [ClassSignature.java:generate].
- `ofLazy(ClassNode)` returns a package-private `Lazy` when a signature exists. `Lazy` does not override `toString()` or `getSuperClass()`, so before `wake()` it renders as `Ljava/lang/Object;` *(observed)*. `wake()` parses once and caches the result. The base `wake()` returns `this` [ClassSignature.java:Lazy].
- `toString()` renders `<T:bound;...>` followed by the superclass and the interfaces. Type variables with an empty bound are omitted. Pending raw interfaces are first moved into the interface list, so `toString()` mutates state [ClassSignature.java:toString].
- `getSuperClass()` returns the raw superclass in descriptor form without type arguments, for example `Ljava/util/AbstractList;` *(observed)* [ClassSignature.java:getSuperClass].
- `addInterface(String internalName)` queues a raw interface. Duplicates of an existing raw type are dropped when the queue is drained [ClassSignature.java:addRawInterface].
- `merge(ClassSignature other)` renames `other`'s type variables so they do not collide with this signature's names, which mutates `other`. It then adds `other`'s type variables and interfaces; the superclass is not merged [ClassSignature.java:merge]:
  - A generic interface replaces a raw one with the same raw type.
  - Renaming tries the next letters cyclically: `T` becomes `U`, then `V`, and so on.
  - For `Z`, the first candidate is `[`, which yields an invalid signature such as `<Z:...;[:Ljava/lang/Object;>` *(observed)* [ClassSignature.java:findOffsetName].
  - Multi-letter names try the patterns `T<name>`, `<name>T`, `TT<name>` and `T<name>Type`.
  - If no name is found, the `IllegalStateException` is printed and the merge is skipped.
- `getRemapper()` returns a `SignatureWriter` that rewrites references to this class's type variables to their current, possibly renamed, names, and leaves locally declared variables alone [ClassSignature.java:SignatureRemapper].
- `TypeVar.equals` compares `currentName` with the other object rather than with that object's name, so two `TypeVar` instances are never equal. The duplicate check in `addTypeVar` is therefore identity-based [ClassSignature.java:TypeVar].

### `Locals` (final, static helpers)

- `Settings(flags, flagsCustom, insnThreshold, frameThreshold)` uses the same thresholds for chopped and trimmed locals. The six-argument form sets them separately. A threshold of -1 means unlimited [Locals.java:Settings].
  - Flags: `RESURRECT_FOR_BOGUS_TOP=1`, `RESURRECT_EXPOSED_ON_LOAD=2`, `RESURRECT_EXPOSED_ON_STORE=4`, `DEFAULT_FLAGS=7`.
  - `DEFAULT = Settings(7, 0, -1, 1, -1, -1)`.
- `getLocalsAt(ClassNode, MethodNode, AbstractInsnNode[, Settings])` returns a `LocalVariableNode[maxLocals]` describing the locals in scope at the node [Locals.java:getLocalsAt]:
  - It first skips forward over up to 3 label or line-number nodes, but never over a frame.
  - Slot 0 is `"this"` (desc `L<owner>;`) for instance methods. Parameters are named `arg0`, `arg1` and so on, at their real slot indices.
  - The second slot of a wide type is `null`, as are slots that hold a `SyntheticLocalVariableNode` or have a `null` desc.
  - It throws `LVTGeneratorError` when `ClassInfo.forName` or `findMethod` fails, or when it meets an unrecognised frame entry. It throws `IllegalStateException("Locals entered an invalid state ...")` when the computed frame shrinks below the argument frame.
- `getLocalVariableAt(ClassNode, MethodNode, AbstractInsnNode, int var)` [Locals.java:getLocalVariableAt]:
  - It returns the LVT entry for `var` whose `[start, end)` range covers the node.
  - Failing that, it consults the generated table (only when the method has a real LVT), and finally returns an out-of-range entry with the same index, or `null`.
- `getLocalVariableTable(ClassNode, MethodNode)` returns an unmodifiable view of `method.localVariables`. When that list is empty, it returns the generated table [Locals.java:getLocalVariableTable].
- `getGeneratedLocalVariableTable(ClassNode, MethodNode)` caches results under the key `"<class>.<name><desc>"` [Locals.java:getGeneratedLocalVariableTable]. The first call returns an unmodifiable view, and later calls return the cached mutable list.
- `generateLocalVariableTable(ClassNode, MethodNode)` [Locals.java:generateLocalVariableTable]:
  - It runs an `Analyzer` with `MixinVerifier`, names the entries `var<N>`, and records `null`-typed values as `Ljava/lang/Object;`.
  - It inserts new `LabelNode`s into `method.instructions`.
  - An `AnalyzerException` is printed and swallowed.
- `loadLocals(Type[] locals, InsnList, int pos, int limit)` appends `xLOAD i` for each non-`null` `locals[i]` from `pos` on, up to `limit` loads [Locals.java:loadLocals].
- `computeFrameSize(FrameNode, int initial)` counts `LONG` and `DOUBLE` as 2 and everything else as 1, and returns `max(initial, size)`. A `null` `local` list gives `initial` [Locals.java:computeFrameSize].
- `getFrameTypeName(Object)` [Locals.java:getFrameTypeName]:
  - `null` gives `NULL`, and a `String` gives its simple name (`java/lang/Object` gives `Object`).
  - Integers 0 to 6 give `TOP`, `INTEGER`, `FLOAT`, `DOUBLE`, `LONG`, `NULL` or `UNINITIALIZED_THIS`. Values of 7 or more give `INVALID`, and negative values throw `ArrayIndexOutOfBoundsException`.
  - Anything else gives `?` *(observed)*.
- `SyntheticLocalVariableNode` is a public marker subclass. Its entries are hidden from `getLocalsAt` results [Locals.java:SyntheticLocalVariableNode].

### Parsers: `ConstraintParser`, `Quantifier`, `VersionNumber`, `JavaVersion`

#### `ConstraintParser`

- `ConstraintParser.parse(String)` [ConstraintParser.java:parse]:
  - `null` or `""` gives `Constraint.NONE` (token `*`, range from `MIN_VALUE` to `MAX_VALUE`, and `check` is a no-op).
  - It removes all whitespace, upper-cases with `Locale.ROOT`, and splits on `;` into chained constraints that `check` combines with AND.
  - `";"` gives `NONE`. Whitespace only, or an empty segment such as `A(1);;B(2)`, throws `InvalidConstraintException("Constraint syntax was invalid parsing: ")` *(observed)*.
- `parse(AnnotationNode)` reads the `constraints` value, defaulting to `""` [ConstraintParser.java:parse].
- Grammar: `TOKEN(` *spec* `)`, where the token matches `[A-Z0-9\-_.]+` [ConstraintParser.java:Constraint]. All of the following were observed:

  | Spec | Range |
  |------|-------|
  | `()` | any value |
  | `(n)` | n to n |
  | `(n+)`, `(n-)`, `(n>)`, `(>=n)` | n or greater |
  | `(>n)` | n+1 or greater |
  | `(<n)` | up to n-1 |
  | `(<=n)`, `(n<)` | up to n |
  | `(a-b)` | a to b |
  | `(n+k)` | n to n+k |

- `InvalidConstraintException` cases *(observed)*:
  - Text that does not match the grammar gives "Constraint syntax was invalid parsing: X".
  - `(5-1)` gives "Invalid range specified '1' is less than 5 in ...".
  - `(>5-6)` gives "Unexpected modifier '>' in ... parsing range".
  - `(>5-)` gives "Unexpected trailing modifier '-' in ...".
  - Numbers too large for an int throw an unwrapped `NumberFormatException`.
- `Constraint.check(ITokenProvider)` [ConstraintParser.java:check]:
  - It throws `ConstraintViolationException` when the token resolves to `null` ("could not be resolved"), with no bad value. When the value is below the minimum or above the maximum, the exception carries the bad value.
  - It then checks the chained constraints in turn.
- `getToken()`, `getMin()`, `getMax()` [ConstraintParser.java:Constraint].
- `getRangeHumanReadable()` gives `ANY VALUE`, `less than or equal to M`, `greater than or equal to N`, `N` or `between N and M` [ConstraintParser.java:getRangeHumanReadable].
- `toString()` gives `Constraint(TOKEN [min-max])` for the first constraint only [ConstraintParser.java:Constraint].

#### `Quantifier`

- `Quantifier.parse(String)` *(observed)* [Quantifier.java:parse]:
  - `null` or blank gives `DEFAULT` (0, -1), `*` gives `ANY` (0, `MAX_VALUE`), and `+` gives `PLUS` (1, `MAX_VALUE`).
  - `{n}` gives (n, n), `{n,}` gives (n, `MAX`), `{,m}` gives (0, m), and `{n,m}` gives (n, m) with no ordering check, so `{5,2}` gives (5, 2).
  - Malformed text, including `{}`, `{a}` and a bare `3`, gives `NONE` (0, 0).
- `getClampedMin()` returns `max(0, min)`. `getClampedMax()` returns 1 when `max < 0`, and `max(min, max)` otherwise [Quantifier.java:getClampedMax].
- `isDefault()` returns `min == 0 && max < 0` [Quantifier.java:isDefault].
- `toString()` *(observed)* [Quantifier.java:toString]:
  - It returns `""` when `max < min` (as for `DEFAULT`) and for (0, 1), which is `SINGLE`.
  - (0, `MAX`) gives `*`, (1, `MAX`) gives `+`, and (n, n) gives `{n}`.
  - Otherwise it returns `{n,m}`, `{n,}` or `{,m}`.
- `equals` matches another `Quantifier` with the same min and max, or a `Number` equal to both min and max. `hashCode` is `31*min*max` [Quantifier.java:equals].

#### `VersionNumber`

- `VersionNumber.parse(String[, String default])` matches `^\d{1,5}(\.\d{1,5}){0,3}(-[A-Za-z0-9_\-]+)?$` [VersionNumber.java:parse]:
  - `null` or a non-matching string gives the default (`NONE`, which renders `0.0`).
  - A part above 32767 throws `IllegalArgumentException("Version parts cannot exceed 32767, found N")`.
  - The `String` default is parsed eagerly.
- Getters are `getMajor()`, `getMinor()`, `getPatch()` (third part), `getRevision()` (fourth part) and `getSuffix()` (for example `"-SNAPSHOT"`, or `""`) [VersionNumber.java:getMajor].
- `toString()` *(observed)* [VersionNumber.java:toString]:
  - It always prints `major.minor`, then `.patch` when patch or revision is non-zero, then `.revision` when that is non-zero, then the suffix.
  - So `1` gives `1.0`, `1.2.0.0` gives `1.2`, and `1.2.0.4` gives `1.2.0.4`.
- `compareTo`, `equals` and `hashCode` use only the packed numeric value and ignore the suffix, so `1.2-SNAPSHOT` equals `1.2`. Comparison is numeric (`1.10` > `1.9`), and `compareTo(null)` returns 1 *(observed)* [VersionNumber.java:compareTo].

#### `JavaVersion`

- `JavaVersion.current()` parses `System.getProperty("java.version")` and caches the result in a static field [JavaVersion.java:current]:
  - It takes the first `\d+\.\d+` match (`1.8.0_292` gives 1.8, and `16.0.2` gives 16.0).
  - Failing that, it takes the first integer.
  - Failing that too, it returns 1.6.
- Constants run from `JAVA_6 = 1.6` through `JAVA_21 = 21.0`, with `JAVA_9` and later written as whole numbers [JavaVersion.java:JAVA_6].

### `PrettyPrinter`

#### Construction and content

- `PrettyPrinter()` uses width 100 and wrap width 80. `PrettyPrinter(int w)` sets the width to `max(40, w)` and the wrap width to `width - 20` [PrettyPrinter.java:PrettyPrinter].
- `wrapTo(int)` is fluent, and `wrapTo()` returns the current wrap width [PrettyPrinter.java:wrapTo].
- All `add`, `kv`, `hr`, `centre`, `table`, `th`, `tr`, `spacing`, `trace`, `print` and `log` methods return `this` [PrettyPrinter.java:add].
- `add()` adds a blank line. `add(String)` and `add(String format, Object...)` add a line and widen the printer to the line length, with no cap. A `null` string throws `NullPointerException` [PrettyPrinter.java:add].
- `add(Object[] [, format])`, `addIndexed(Object[])` (formats `[%Nd] %s`) and `addWithIndices(Collection)` [PrettyPrinter.java:addIndexed].
- `add(IPrettyPrintable)` accepts `null` [PrettyPrinter.java:add].
- `add(Throwable[, indent])` prints each cause as `    Class: message` followed by its stack frames [PrettyPrinter.java:add].
- `add(Object[, indent])` dispatches by type: `String`, `Iterable`, `Map` (as key-value lines), `IPrettyPrintable`, `Throwable` or array. Watch for these cases [PrettyPrinter.java:append]:
  - An `Object[]` passed as `Object` is printed with the format `indent + "%s"`, so `add((Object) new Object[]{"x"})` prints `0x` *(observed)*.
  - Primitive arrays throw `ClassCastException`, and `null` throws `NullPointerException`.
- `addWrapped([width,] format, args)` [PrettyPrinter.java:addWrapped]:
  - It converts tabs to 4 spaces and reuses the leading whitespace as the continuation indent.
  - It breaks at the last newline, space or tab at or before `width`. When no break point lies at or after index 10, it hard-wraps at `width` and drops the character at that position (`...0123` / `56789...`, losing `4`) *(observed)*.
- `kv(key, value)` and `kv(key, format, args)` add a key-value line [PrettyPrinter.java:kv]:
  - Lines render as `String.format("%<w>s : %s")` with the key right-aligned.
  - `<w>` is the widest key seen so far (at least 10), and it is evaluated when the printer renders.
  - `kvWidth(int)` only ever grows `<w>`. `add(Map)` prints `null` keys as `"null"`.
- `hr()` and `hr(char)` add a rule. `centre()` centres the previous line only when it is a `String` [PrettyPrinter.java:centre].
- Tables [PrettyPrinter.java:table]:
  - `table(String...)` and `table(Object...)` start a table. In the `Object...` form, an `Integer > 0` sets the column width, an `Integer < 0` sets the maximum width, and an `Alignment` sets alignment. Integers and alignments that come before any column are ignored.
  - `th()` adds the header row. `tr(Object...)` adds the header automatically if it is missing, then calls `toString()` on each cell (a `null` cell throws `NullPointerException`).
  - `spacing(int)` sets the column gap, which defaults to 2.

#### Output

- `print()` writes to `System.err`, and `print(PrintStream)` writes to the given stream [PrettyPrinter.java:print]. *(observed)* for width 40:
  - The frame lines are `/*` + `*`×(width+2) + `*/`.
  - Each content line is `/* %-<width>s */`.
  - A rule line is `/*` + char×(width+2) + `*/`.
  - Width grows to fit variable-width entries (key-value pairs and tables), capped at 4096.
- `log(ILogger[, Level])` defaults to `Level.INFO` [PrettyPrinter.java:log].
- `log(Level)` and the `trace(...)` overloads that take a logger name resolve the logger through `MixinService.getService().getLogger(name)`. The default name is the caller's simple class name [PrettyPrinter.java:getDefaultLoggerName].
- `trace(...)` both logs (default `Level.DEBUG`) and prints (default `System.err`) [PrettyPrinter.java:trace].
- The static `dumpStack()` and `print(Throwable)` write to `System.err` [PrettyPrinter.java:dumpStack].

### `SignaturePrinter`

- Constructors take any of the following [SignaturePrinter.java:SignaturePrinter]:
  - `(MethodNode[, String[] argNames])`
  - `(ITargetSelectorByName)`
  - `(String name, String desc)`
  - `(Type[])`
  - `(Type ret, Type[])`
  - `(String, Type, Type[])`, which names arguments `var0`, `var1` and so on, skipping `null` entries
  - `(String, Type, LocalVariableNode[])`
  - `(String, Type, Type[], String[])`
- The `MethodNode` constructors store `returnType = VOID_TYPE` and put the real return type in the modifiers [SignaturePrinter.java:setModifiers]:
  - A `public static (I)Ljava/lang/String;` method `bar` prints `public static String bar(int var0)`.
  - `getReturnType()` gives `void`, and `toDescriptor()` gives `(int)void` *(observed)*.
- `toString()` *(observed)* [SignaturePrinter.java:toString]:
  - It prints the modifiers (default `"private void"`, whatever the descriptor), then the name (`"method"` when `null`), then the arguments.
  - Array arguments put the brackets after the name: `("foo","(I[Ljava/lang/String;J)Ljava/lang/Object;")` gives `private void foo(int var0, String var1[], long var2)`.
  - A `null` first argument type gives a leading `", "`: `(, int var0)`.
  - Missing names print as `unnamed<i>`.
  - A `CallbackInfoReturnable` argument gets the boxed return type, as in `CallbackInfoReturnable<Integer> var0`.
- `toDescriptor()` gives a readable form such as `(int,String[],long)Object`, not a JVM descriptor *(observed)* [SignaturePrinter.java:toDescriptor].
- `getFormattedArgs()` gives types only, such as `(int, Map$Entry)` [SignaturePrinter.java:getFormattedArgs].
- `setModifiers(String)` replaces `${returnType}`. `setFullyQualified(boolean)` and `isFullyQualified()` control qualified names [SignaturePrinter.java:setModifiers].
- `getTypeName(Type[, box[, fullyQualified]])` [SignaturePrinter.java:getTypeName]:
  - `null` gives `{null?}`, a method type gives `Object`, and `[[I` boxed gives `Integer[][]`.
  - The one-argument form is fully qualified, and the `(Type, boolean)` form is not *(observed)*.

### Other helpers

- `Handles` [Handles.java:Handles]:
  - `isField(Handle)` returns true for `H_GET*` and `H_PUT*` tags, false for `H_INVOKE*` and `H_NEWINVOKESPECIAL`, and throws `IllegalArgumentException("Invalid tag ...")` for anything else.
  - `opcodeFromTag(int)` maps tags 1 to 9 to opcodes and returns 0 when out of range. `H_NEWINVOKESPECIAL` (8) maps to `INVOKESPECIAL`.
  - `tagFromOpcode(int)` returns the first matching tag, so `INVOKESPECIAL` gives 7 and never 8. An unknown opcode gives 0 *(observed)*.
- `ObfuscationUtil` [ObfuscationUtil.java:remapDescriptor]:
  - `mapDescriptor` and `unmapDescriptor(desc, IClassRemapper)` rewrite every `L...;` token through `map` or `unmap`. When the remapper returns `null`, the name is kept.
  - Primitives and `[` pass through, so `(La/B;[La/B;ILx/Y;)La/B;` becomes `(Lc/D;[Lc/D;ILx/Y;)Lc/D;` *(observed)*.
  - A missing `;` throws `IllegalArgumentException("Invalid descriptor '...', missing ';'")`.
- `LanguageFeatures` [LanguageFeatures.java:scan]:
  - Bit values: `METHODS_IN_INTERFACES=1`, `PRIVATE_SYNTHETIC_METHODS_IN_INTERFACES=2`, `PRIVATE_METHODS_IN_INTERFACES=4`, `NESTING=8`, `DYNAMIC_CONSTANTS=16`, `RECORDS=32`, `SEALED_CLASSES=64`.
  - `scan(ClassNode)` detects `NESTING` (a nest host, or at least one nest member), the interface features (non-abstract methods, and private methods split by the synthetic flag) and `DYNAMIC_CONSTANTS` (an LDC of `ConstantDynamic` in a non-interface method). It never sets `RECORDS` or `SEALED_CLASSES`.
  - `format(int)` lists field names, as in `[NESTING,RECORDS]`, and gives `[]` for 0 *(observed)*.
- `Constants` [Constants.java:Constants]:
  - `CTOR="<init>"`, `CLINIT="<clinit>"`, `IMAGINARY_SUPER="super$"`.
  - `DEBUG_OUTPUT_PATH=".mixin.out"`, with `DEBUG_OUTPUT_DIR` as the matching relative `File`.
  - `MIXIN_PACKAGE="org.spongepowered.asm.mixin"` (from `Mixin.class`) and `MIXIN_PACKAGE_REF="org/spongepowered/asm/mixin"`.
  - `STRING`, `OBJECT`, `CLASS` and their `*_DESC` forms, `SYNTHETIC_PACKAGE="org.spongepowered.asm.synthetic"`, `UNICODE_SNOWMAN='☃'`, and the `SIDE_*` strings.
  - `ManifestAttributes`: `TweakClass`, `Main-Class`, `MixinConfigs`, `MixinTokenProviders`, `MixinConnector`, and the deprecated `MixinCompatibilityLevel`.
- `Files` [Files.java:toFile]:
  - `toFile(URL|URI)` returns `null` for `null`. A `file://host/...` URI is rewritten to `file:////host/...` (UNC) before `new File(uri)`, which throws `IllegalArgumentException` for a URI that is not a `file` URI.
  - `toFile(URL)` declares `URISyntaxException`.
  - `deleteRecursively(File)` does nothing for `null` or for a path that is not a directory. It throws `IOException` when it cannot list or delete something, and wraps `SecurityException` in `IOException` [Files.java:deleteRecursively].
- `ReEntranceLock(int maxDepth)` [ReEntranceLock.java:ReEntranceLock]:
  - `push()` increments the depth, then runs `checkAndSet()`, which sets a sticky semaphore when the depth exceeds `maxDepth`.
  - `pop()` at depth 0 throws `IllegalStateException("ReEntranceLock pop() with zero depth")`.
  - `check()` tests only the current depth. `isSet()`, `set()` and `clear()` work on the semaphore, and popping does not clear it *(observed)*.
- `Counter` has a public mutable `int value`. `equals` holds only for exact class `Counter` with the same value, and `hashCode == value` [Counter.java:Counter].
- Interfaces:
  - `IConsumer<T>.accept(T)` [IConsumer.java:accept].
  - `IMessageSink.addMessage(String format, Object...)` [IMessageSink.java:addMessage].
  - `ITokenProvider.getToken(String)` returns an `Integer`, or `null` when the token is not provided [ITokenProvider.java:getToken].

## Invariants

- The static-only classes cannot be instantiated: each has a private constructor. `Bytecode`, `Annotations`, `Locals`, `ConstraintParser`, `Handles`, `Files` and `LanguageFeatures` are final, and `Constants`, `ObfuscationUtil` and `JavaVersion` are abstract [Bytecode.java:Bytecode] [Constants.java:Constants].
- Instances come from these sources:
  - `VersionNumber` has only private constructors, so instances come from `parse` or `NONE` [VersionNumber.java:parse].
  - `Quantifier(int, int)` and `Counter` (implicit) have public constructors [Quantifier.java:Quantifier] [Counter.java:Counter].
  - `ClassSignature`'s constructor is package-private, so instances come from `of` or `ofLazy` [ClassSignature.java:of].
- `VersionNumber` packs four non-negative shorts into a `long` (major in bits 48 and up). Ordering and equality depend only on that `long` [VersionNumber.java:pack].
- `Constraint` ranges are inclusive, from `min` to `max`. An unbounded side uses `Integer.MIN_VALUE` or `Integer.MAX_VALUE` [ConstraintParser.java:Constraint].
- `Bytecode.getOpcodeName` and `parseOpcodeName` rely on the reflection order of `org.objectweb.asm.Opcodes` fields, starting at `UNINITIALIZED_THIS`. `LanguageFeatures.format` relies on the declared-field order of its own class [Bytecode.java:getOpcodeName] [LanguageFeatures.java:format].
- `Locals.getLocalsAt` always returns an array of length `method.maxLocals`. When an entry is a wide type, the slot after it is `null` [Locals.java:getLocalsAt].
- `ClassSignature` keeps type variables in insertion order (`LinkedHashMap`). `merge` never changes the receiver's superclass [ClassSignature.java:merge].

## Side Effects

- Global static mutable state:
  - `Locals.calculatedLocalVariables` is a plain `HashMap` cache, never cleared and not synchronised [Locals.java:getGeneratedLocalVariableTable].
  - `JavaVersion.current` is cached on first call [JavaVersion.java:current].
  - The `Quantifier` constants `DEFAULT`, `NONE`, `SINGLE`, `ANY` and `PLUS` are `public static` and not `final` [Quantifier.java:DEFAULT].
  - The `Locals.Settings` flags and `Settings.DEFAULT` are `public static` and not `final` [Locals.java:Settings].
  - The public `Bytecode.CONSTANTS_*` arrays are mutable [Bytecode.java:CONSTANTS_INT].
  - `Annotations.mergeableAnnotationPattern` is computed once at class initialisation [Annotations.java:getMergeableAnnotationPattern].
- Node mutation:
  - `Locals.generateLocalVariableTable` inserts `LabelNode`s into the method [Locals.java:generateLocalVariableTable].
  - The `Annotations` enum-list `getValue` rewrites the annotation's value list [Annotations.java:getValue].
  - `Annotations.setValue`, `setVisible` and `setInvisible` modify the node they are given [Annotations.java:setValue].
  - `Annotations.merge` and `Bytecode.merge`/`replace`/`setVisibility` modify the destination node [Bytecode.java:merge].
  - `ClassSignature.merge` renames type variables on its argument, and `toString` drains queued interfaces [ClassSignature.java:merge].
- Console output:
  - `Bytecode.printMethod`, `printMethodWithOpcodeIndices` and `printNode` write to `System.err`, and `Bytecode.dumpClass` writes to `System.out` [Bytecode.java:printMethod].
  - `PrettyPrinter.print()`, `dumpStack()` and `print(Throwable)` write to `System.err` [PrettyPrinter.java:print].
  - `ClassSignature.read` and `merge`, and `Locals.generateLocalVariableTable`, print swallowed exceptions to stderr [ClassSignature.java:read].
- Service and metadata dependencies, which need a bootstrapped `org.spongepowered.asm.service.MixinService` or `org.spongepowered.asm.mixin.transformer.ClassInfo`:
  - `Locals.getLocalsAt` uses `ClassInfo.forName` [Locals.java:getLocalsAt].
  - `Locals.generateLocalVariableTable` resolves types through `MixinVerifier` and `ClassInfo` [Locals.java:generateLocalVariableTable].
  - `Annotations.merge` uses the service's logger, but only on error [Annotations.java:merge].
  - `PrettyPrinter.log(Level)` and the `trace(...)` overloads without an explicit `ILogger` use the service [PrettyPrinter.java:trace].
- Filesystem: `Files.deleteRecursively` deletes files and directories [Files.java:deleteRecursively].

## Verification

Run these from the repository root:

- `mise exec -- ./gradlew compileJava`
- `mise exec -- ./gradlew test`

`build.gradle` declares a `test` source set, but `src/test` does not exist yet, so there are no tests today.

Pure, service-free unit-test targets:

- `Bytecode`: descriptors, constants, visibility, opcode names, `loadArgs`, `compareBridgeMethods`, `merge`
- `VersionNumber`, `Quantifier`, `ConstraintParser`
- `SignaturePrinter`, `PrettyPrinter` (via `print(PrintStream)`)
- `ObfuscationUtil`, `Handles`, `ReEntranceLock`
- `ClassSignature` (`of`, `merge`, `toString`)
- `Annotations`: get, set and value helpers
- `LanguageFeatures`, `Locals.computeFrameSize` and `getFrameTypeName`

These need `ClassInfo` or `MixinService` and suit integration tests: `Locals.getLocalsAt` and `generateLocalVariableTable`, and logger-based `PrettyPrinter` output.
