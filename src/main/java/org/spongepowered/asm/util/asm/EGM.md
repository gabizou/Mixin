# org.spongepowered.asm.util.asm

## Overview

Package `org.spongepowered.asm.util.asm` holds Mixin's helpers for working with the ASM library. It covers runtime detection of the ASM version on the classpath, access to nest-host and nest-member fields across ASM versions, two annotation abstractions, and ASM subclasses: a marker label, a method node that remembers its owner, a method visitor that can emit constants, and a bytecode verifier that resolves types through Mixin's `ClassInfo`. The main source set compiles at Java 1.6 against ASM 9.5 (`gradle.properties: asmVersion=9.5`).

## Interface

### `ASM` (final, not instantiable)

All state is static. It is computed once, during class initialisation, by `detectVersion()` [ASM.java:detectVersion].

- `public static final int API_VERSION`: the raw `Opcodes.ASM*` value of the highest non-experimental `ASM*` field in `org.objectweb.asm.Opcodes`. If no such field qualifies, it stays at `Opcodes.ASM4` [ASM.java:API_VERSION].
- `isAtLeastVersion(int major)` returns `majorVersion >= major` [ASM.java:isAtLeastVersion(int)].
- `isAtLeastVersion(int major, int minor)` returns `majorVersion > major || (majorVersion == major && implMinorVersion >= minor)`. It compares against the implementation minor version, not the API minor version that `getApiVersionMinor()` returns [ASM.java:isAtLeastVersion(int,int)].
- `isAtLeastVersion(int major, int minor, int patch)`: when `majorVersion == major`, the result is `implMinorVersion > minor || (implMinorVersion == minor && patchVersion >= patch)`. Otherwise the result is `majorVersion > major` [ASM.java:isAtLeastVersion(int,int,int)].
- `getApiVersionMajor()` and `getApiVersionMinor()` return the major and minor bytes of the chosen `ASM*` constant [ASM.java:getApiVersionMajor] [ASM.java:getApiVersionMinor].
- `getApiVersionString()` returns `String.format("%d.%d", major, minor)` [ASM.java:getApiVersionString].
- `getVersionString()` returns `"ASM %d.%d%s (%s)"` formatted with the major version, the implementation minor version, `"." + patch` (only when the patch is above 0) and `maxVersion`. `maxVersion` is the name of the last `ASM*` field whose major is at least the current major, and that field can be experimental. It defaults to `"FALLBACK"` [ASM.java:getVersionString].
- `getMaxSupportedClassVersion()` returns the raw `Opcodes.V*` value (`minor << 16 | major`). `getMaxSupportedClassVersionMajor()` and `getMaxSupportedClassVersionMinor()` return its low and high 16 bits. The default is `Opcodes.V1_6` [ASM.java:getMaxSupportedClassVersion].
- `getClassVersionString()` returns `"Up to Java %s (class file version %d.%d)"`. A detected Java version is the field name with the leading `V` removed and `_` replaced by `.`, so `V1_8` becomes `1.8` and `V17` becomes `17`. The undetected default is the literal `"V1.6"`, which keeps its `V` [ASM.java:getClassVersionString].

### `ClassNodeAdapter` (final, not instantiable)

These helpers read and write `ClassNode` nest attributes. With ASM 7 or later (true for the ASM 9.5 build), every method uses the public fields `classNode.nestHostClass` and `classNode.nestMembers` directly. On older ASM versions they use reflection on `nestHostClass` or `nestHostClassExperimental` and on `nestMembers` or `nestMembersExperimental` [ClassNodeAdapter.java:getField].

- `getNestHostClass(ClassNode)` returns the nest-host internal name or `null`. A `null` node throws `NullPointerException` [ClassNodeAdapter.java:getNestHostClass].
- `setNestHostClass(ClassNode, String)` assigns the field. With ASM 7 or later it does not return early, so it may also write the value again through reflection [ClassNodeAdapter.java:setNestHostClass].
- `getNestMembers(ClassNode)` returns the live `nestMembers` list, which may be `null` [ClassNodeAdapter.java:getNestMembers].
- `getNestMembersAsList(ClassNode)` never returns `null`. If the node has no list, it creates a new `ArrayList`, installs it with `setNestMembers` and returns it [ClassNodeAdapter.java:getNestMembersAsList].
- `setNestMembers(ClassNode, List<String>)` assigns the list and, with ASM 7 or later, returns early [ClassNodeAdapter.java:setNestMembers].

### `IAnnotatedElement` and `IAnnotationHandle` (interfaces)

These interfaces have no implementations in this directory. `org.spongepowered.asm.util.Annotations.Handle` in the parent package implements `IAnnotationHandle`.

- `IAnnotatedElement.getAnnotation(Class<? extends Annotation>)` returns an `IAnnotationHandle` [IAnnotatedElement.java:getAnnotation].
- `IAnnotationHandle` declares `exists()`, which is false when the wrapped element is `null`. It also declares `getDesc()`, `getAnnotationList(key)`, `getTypeValue(key)` (class literals as `org.objectweb.asm.Type`), `getTypeList(key)` and `getAnnotation(key)`, which returns `null` when the key is unset. The remaining methods are `getValue(key, default)` (returns the default when the key is unset), `getValue()`, `getValue(key)` (`null` when unset), `getBoolean(key, default)`, `getList()` and `getList(key)` (an empty list when unset) [IAnnotationHandle.java:interface].

### `MarkerNode extends LabelNode`

- `public static final int INITIALISER_TAIL = 1` and `public static final int BODY_START = 2`. The public final field `type` is not validated, so the constructor accepts any int [MarkerNode.java:class].
- `MarkerNode(int type)` calls `super(null)` [MarkerNode.java:MarkerNode(int)].
- `accept(MethodVisitor)` does nothing, so the marker is never written to a visitor [MarkerNode.java:accept].

### `MethodNodeEx extends MethodNode`

- `MethodNodeEx(int access, String name, String desc, String signature, String[] exceptions, IMixinInfo owner)` calls `super(ASM.API_VERSION, ...)` and saves `name` as `originalName` [MethodNodeEx.java:MethodNodeEx].
- `toString()` returns `originalName + desc`. `getQualifiedName()` returns `owner.getName() + "::" + originalName` and throws `NullPointerException` when `owner` is `null` [MethodNodeEx.java:getQualifiedName].
- `getOriginalName()` and `getOwner()` return the values passed to the constructor [MethodNodeEx.java:getOriginalName].
- `static getName(MethodNode)` returns `getOriginalName()` for a `MethodNodeEx` and `method.name` for any other node. A `null` argument throws `NullPointerException` [MethodNodeEx.java:getName].

### `MethodVisitorEx extends MethodVisitor`

- `MethodVisitorEx(MethodVisitor mv)` calls `super(ASM.API_VERSION, mv)` [MethodVisitorEx.java:class].
- `visitConstant(byte c)` emits `visitInsn(Bytecode.CONSTANTS_INT[c + 1])` (`ICONST_M1` to `ICONST_5`) for values from -1 to 5. For every other value it emits `visitIntInsn(BIPUSH, c)`. The boundaries are -2, -1, 5 and 6 [MethodVisitorEx.java:visitConstant].

### `MixinVerifier extends SimpleVerifier`

- `MixinVerifier(int api, Type currentClass, Type currentSuperClass, List<Type> currentClassInterfaces, boolean isInterface)` stores its arguments without null checks [MixinVerifier.java:MixinVerifier].
- `isInterface(Type)` returns the constructor flag for `currentClass`. For any other type it returns `ClassInfo.forType(type, TypeLookup.ELEMENT_TYPE).isInterface()`, which throws `NullPointerException` when the type cannot be resolved [MixinVerifier.java:isInterface].
- `getSuperClass(Type)` returns `currentSuperClass` for `currentClass`. For any other type it asks `ClassInfo` and returns `null` when there is no superclass [MixinVerifier.java:getSuperClass].
- `isAssignableFrom(Type type, Type other)` returns true for equal types and handles `currentClass` on either side specially. An interface target is widened to `java/lang/Object`. In all other cases it returns `ClassInfo.forType(other, ELEMENT_TYPE).hasSuperClass(typeInfo)`, and it returns false when `type` cannot be resolved [MixinVerifier.java:isAssignableFrom].

## Invariants

- The static fields of `ASM` start as `majorVersion=5`, `minorVersion=0`, `implMinorVersion=0`, `patchVersion=0`, `maxVersion="FALLBACK"`, `maxClassVersion=Opcodes.V1_6` and `maxJavaVersion="V1.6"`. `detectVersion()` changes them once, and nothing writes them afterwards [ASM.java:static-init].
- Because the initial major version is 5, an `ASM4` field never updates the state. Experimental `ASM*` constants (non-zero bits 24 and up) only update `maxVersion` [ASM.java:detectVersion].
- `detectVersion()` takes the minor and patch versions from the package (`Package.getImplementationVersion()`, or else the jar manifest through `org.spongepowered.asm.launch.platform.MainAttributes`) only when the package major equals the field major and the field minor byte is 0. Otherwise `implMinorVersion == minorVersion` and `patchVersion == 0` [ASM.java:detectVersion].
- `detectVersion()` stops scanning `Opcodes` fields at the field named `ACC_PUBLIC`. A `V*` constant replaces the maximum only when it is strictly greater (major first, then minor) [ASM.java:detectVersion].
- Nothing reaches the reflective `ClassNodeAdapter` path while `ASM.isAtLeastVersion(7)` is true [ClassNodeAdapter.java:getNestHostClass].
- `MethodNodeEx.originalName` is final. Later changes to `name` do not affect `toString`, `getOriginalName` or `getQualifiedName` [MethodNodeEx.java:originalName].

## Side Effects

- Initialising the `ASM` class reflects over `org.objectweb.asm.Opcodes` and may read the jar manifest of the ASM code source. A `ReflectiveOperationException` is rethrown as `java.lang.Error` [ASM.java:detectVersion].
- Initialising `ClassNodeAdapter` reflectively looks up `ClassNode` fields. After any `ReflectiveOperationException`, the global static `notSupported` flag is permanently set to true. The flag is not synchronised [ClassNodeAdapter.java:notSupported].
- `setNestHostClass`, `setNestMembers` and `getNestMembersAsList` mutate the `ClassNode` passed in [ClassNodeAdapter.java:getNestMembersAsList].
- Constructing a `MethodNodeEx` or `MethodVisitorEx` triggers `ASM` class initialisation through `ASM.API_VERSION` [MethodNodeEx.java:MethodNodeEx].
- `MixinVerifier` calls the global `org.spongepowered.asm.mixin.transformer.ClassInfo` cache (`forType`, `forName`), which may load classes through the active `MixinService`. Unit tests therefore need a working service, unless every type compared is `currentClass` or equal [MixinVerifier.java:isAssignableFrom].

## Verification

Run these from the repository root:

- `mise exec -- ./gradlew compileJava`
- `mise exec -- ./gradlew test`

`build.gradle` declares a `test` source set, but `src/test` does not exist yet, so there are no tests today. Good unit-test targets are `MethodVisitorEx.visitConstant` (boundaries -2, -1, 5 and 6), `MarkerNode.accept`, `MethodNodeEx` naming, `ClassNodeAdapter` on the ASM 7+ path, and the formats of the `ASM` version strings.
