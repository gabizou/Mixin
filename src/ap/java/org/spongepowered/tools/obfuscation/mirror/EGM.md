# mirror

## Overview

Wrappers around the `javax.lang.model` mirror API, and ASM fallbacks, that let
the annotation processor examine mixin targets. A target may be a source type,
a classpath class, a type that cannot be resolved (an imaginary type), or the
simulated type of a `@Pseudo` mixin. The package also contains `TypeUtils`,
which converts mirror types to bytecode descriptors and Java-style signatures.
`AnnotatedMixins.getTypeHandle` in
`src/ap/java/org/spongepowered/tools/obfuscation/AnnotatedMixins.java`
chooses which handle class to create.

## Interface

- `AnnotationHandle`: public final class that implements `IAnnotationHandle`.
  - Factories: `of(AnnotationMirror)`, `of(Element, Class<? extends Annotation>)`
    and the constant `MISSING`.
  - Accessors: `asMirror()`, static `asMirror(IAnnotationHandle)`, `exists()`
    and `getDesc()`.
  - Value readers: `getValue()`, `getValue(key)`, `getValue(key, default)`,
    `getBoolean(key, default)`, `getAnnotation(key)`, `getList()`,
    `getList(key)`, `getAnnotationList(key)`, `getTypeValue(key)` and
    `getTypeList(key)`.
  [AnnotationHandle.java:AnnotationHandle]
- `MemberHandle<T extends IMapping<T>>`: public abstract class with
  `getOwner()`, `getName()`, `getDesc()`, abstract `getVisibility()` and
  abstract `asMapping(boolean includeOwner)` [MemberHandle.java:MemberHandle].
- `FieldHandle extends MemberHandle<MappingField>`: `isImaginary()`,
  `getElement()`, `getVisibility()`, `isRawType()` and `asMapping(boolean)`
  [FieldHandle.java:FieldHandle].
- `FieldHandleASM extends FieldHandle`: backed by an ASM `FieldNode`
  [FieldHandleASM.java:FieldHandleASM].
- `MethodHandle extends MemberHandle<MappingMethod>`: `isImaginary()`,
  `getElement()`, `getJavaSignature()`, `getVisibility()` and
  `asMapping(boolean)` [MethodHandle.java:MethodHandle].
- `MethodHandleASM extends MethodHandle`: backed by an ASM `MethodNode`
  [MethodHandleASM.java:MethodHandleASM].
- `TypeHandle`: public class.
  - Constructors: `(PackageElement, String, ITypeHandleProvider)` creates an
    imaginary handle. `(TypeElement, ITypeHandleProvider)` and
    `(DeclaredType, ITypeHandleProvider)` create real handles.
  - Methods: `getName`, `getSimpleName`, `getPackage`, `getElement`,
    `getAnnotation`, `hasTypeMirror`, `getTypeMirror`, `getSuperclass`,
    `getInterfaces`, `getMethods`, `isPublic`, `isImaginary`, `isSimulated`,
    `isNotInterface`, `isSuperTypeOf`, `getReference`, `getMappingMethod`,
    `findDescriptor`, `findField` and `findMethod`.
  [TypeHandle.java:TypeHandle]
- `TypeHandleASM extends TypeHandle`: has the static factory
  `of(PackageElement, String, IMixinAnnotationProcessor)` [TypeHandleASM.java:of].
- `TypeHandleSimulated extends TypeHandle`: constructors
  `(String, TypeMirror, ITypeHandleProvider)` and
  `(PackageElement, String, TypeMirror, ITypeHandleProvider)` [TypeHandleSimulated.java:TypeHandleSimulated].
- `TypeReference`: public class that implements `Serializable` and
  `Comparable<TypeReference>`. Methods are `getName()`, `getClassName()` and
  `getHandle(ITypeHandleProvider)` [TypeReference.java:TypeReference].
- `TypeUtils`: public abstract class with static methods only.
  - Package and naming: `getPackage`, `getElementType`, `stripGenerics`,
    `getName`, `getJavaSignature(Element)`, `getJavaSignature(String)`,
    `getSimpleName`, `getTypeName`, `getDescriptor` and `getInternalName`.
  - Type checks: `isAssignable`, `isEquivalentType` and `getVisibility`.
  - Nested types: enum `Equivalency` and class `EquivalencyResult`.
  [TypeUtils.java:TypeUtils]
- Subpackage [mapping](./mapping/EGM.md): `MappingMethodResolvable`, a method
  mapping that searches up the hierarchy.

## Invariants

### AnnotationHandle

- `getValue(key, default)` reads only values set explicitly in source
  (`AnnotationMirror.getElementValues()`). Defaults declared on the annotation
  type are not read, so callers supply their own defaults. A missing
  annotation also returns the default [AnnotationHandle.java:getValue].
- When the default is an `Enum` and a value is present, the enum constant is
  resolved by the simple name of the `VariableElement` [AnnotationHandle.java:getValue].
- `getList` and `getAnnotationList` return an empty list when the key is
  unset. `getAnnotationList` also accepts a single bare `AnnotationMirror`,
  which JDT produces [AnnotationHandle.java:getAnnotationList].
- `getTypeValue` returns `Type.VOID_TYPE` when the key is unset [AnnotationHandle.java:getTypeValue].
- `toString()` returns `@SimpleName`. For a missing annotation it returns
  `@{UnknownAnnotation}`, and `getDesc()` returns `java/lang/Annotation`
  [AnnotationHandle.java:toString].
- `of(Element, Class)` matches the annotation type by qualified name. If the
  annotation or element is absent, it returns a handle around `null`, for
  which `exists()` is false [AnnotationHandle.java:of].

### TypeHandle (mirror-backed or imaginary)

- `getName()` uses internal form with `/` separators. `toString()` uses `.`
  separators [TypeHandle.java:toString].
- `isImaginary()` is true when there is no target `TypeElement`. Imaginary
  handles return `null` from `getSuperclass()` and empty lists from member
  queries [TypeHandle.java:isImaginary].
- `findField(name, type, matchCase)` matches the field name (exactly, or
  ignoring case when `matchCase` is false). It also requires the type to match:
  the element's Java type, the element's type without generics, or an empty
  string. If the generic query type fails, the type with generics removed is
  tried, and the result has `isRawType() == true` [TypeHandle.java:findField].
- `findMethod` searches both `CONSTRUCTOR` and `METHOD` elements and compares
  Java signatures in the form `(arg.Type,int)ret.Type`, with or without
  generics [TypeHandle.java:findMethod].
- `findDescriptor(selector)` returns the selector's descriptor when it has
  one. Otherwise it returns the descriptor of the first `METHOD` (never a
  constructor) with the same name, or `null` [TypeHandle.java:findDescriptor].
- `isPublic()` requires the type and every enclosing type to be `public`
  [TypeHandle.java:isPublic].
- `isSuperTypeOf(other)` walks the superclasses and interfaces of `other`
  recursively, comparing internal names [TypeHandle.java:isSuperTypeOf].
- `getMappingMethod(name, desc)` returns a `MappingMethodResolvable` owned by
  this handle [TypeHandle.java:getMappingMethod].

### TypeHandleASM (classpath bytecode)

- `of` reads `<name>.class` with `Filer.getResource(StandardLocation.CLASS_PATH, pkg, ...)`.
  On `FileNotFoundException` it falls back to `PLATFORM_CLASS_PATH`, then
  parses the bytes with ASM `ClassReader.accept(classNode, 0)` [TypeHandleASM.java:of].
- Results are cached in a static map keyed by the dotted, fully qualified
  name. A `FileNotFoundException` caches `null`. Other exceptions return `null`
  without caching [TypeHandleASM.java:cache].
- `isImaginary()` and `hasTypeMirror()` both return false, and
  `getTypeMirror()` returns `null` [TypeHandleASM.java:isImaginary].
- `getMethods()` skips names that start with `<` and synthetic methods
  [TypeHandleASM.java:getMethods].
- `isPublic()` and `isNotInterface()` read the class access flags
  [TypeHandleASM.java:isPublic].
- `findMethod` and `findField` compare the name and the Java signature built
  from the bytecode descriptor. Unlike `TypeHandle`, they have no fallback that
  removes generics [TypeHandleASM.java:findMethod].
- `getAnnotation` reads visible annotations first, then invisible ones
  [TypeHandleASM.java:getAnnotation].

### TypeHandleSimulated (`@Pseudo` targets)

- The target element is the mixin's own type. `isSimulated()` is true,
  `isImaginary()` is false and `isPublic()` is true [TypeHandleSimulated.java:isSimulated].
- `getAnnotation` returns `null` rather than a missing handle, and
  `getSuperclass` returns `null` [TypeHandleSimulated.java:getAnnotation].
- `findDescriptor` returns the selector's own descriptor, which may be
  `null` [TypeHandleSimulated.java:findDescriptor].
- `findField` and `findMethod` always return a new handle with no owner. They
  never return `null` [TypeHandleSimulated.java:findMethod].
- `getMappingMethod` searches the mixin's hierarchy for a matching method,
  interfaces before the superclass, and returns `asMapping(true)` for a match.
  Otherwise it falls back to `TypeHandle.getMappingMethod` [TypeHandleSimulated.java:getMappingMethod].

### Member handles and TypeReference

- `MethodHandle.asMapping(true)` returns a `MappingMethodResolvable` when it
  has an owner handle. `asMapping(false)` returns a mapping with a `null`
  owner [MethodHandle.java:asMapping].
- `toString()` formats are `L<owner>;<name><desc>` for methods and
  `L<owner>;<name>:<desc>` for fields [FieldHandle.java:toString].
- `getVisibility()` on a mirror handle with no element returns `null`
  [TypeUtils.java:getVisibility].
- Only `TypeReference.name` is serialized. The handle is transient and
  resolved again on demand. Equality, hash code and ordering use the name
  alone [TypeReference.java:equals].

### TypeUtils conversions

- `getInternalName(TypeMirror)` maps types to descriptors as follows. Any
  other kind throws `IllegalArgumentException("Unable to parse type symbol ...")`
  [TypeUtils.java:getInternalName].
  - Arrays become `[` followed by the component descriptor.
  - Declared types become `L<internal>;`.
  - Type variables become the descriptor of their upper bound.
  - Primitives become `Z B C D F I J S V`.
  - `ERROR` types become the `Object` descriptor.
- `getInternalName(TypeElement)` joins nested types with `$` and packages
  with `/` [TypeUtils.java:getInternalName].
- `getDescriptor(ExecutableElement)` returns `(<params>)<ret>`
  [TypeUtils.java:getDescriptor].
- `getJavaSignature(Element)` returns `(T1,T2)R` with dotted type names
  [TypeUtils.java:getJavaSignature].
- `getTypeName` writes arrays as `T[]` and maps `ERROR` types to
  `java.lang.Object` [TypeUtils.java:getTypeName].
- `stripGenerics` removes every `<...>` section, including nested ones
  [TypeUtils.java:stripGenerics].
- Upper-bound resolution stops after 5 levels and then throws
  `IllegalArgumentException("Type symbol \"<type>\" is too complex")`
  [TypeUtils.java:getUpperBound].
- `getVisibility` returns the first `PUBLIC`, `PROTECTED` or `PRIVATE`
  modifier it finds, or `PACKAGE` if there is none [TypeUtils.java:getVisibility].
- `isAssignable` retries with raw types when the generic check fails
  [TypeUtils.java:isAssignable].
- `isEquivalentType` returns one of `EQUIVALENT`, `EQUIVALENT_BUT_RAW` (with
  `rawType` set to 1 or 2), `BOUNDS_MISMATCH` or `NOT_EQUIVALENT`, plus a
  readable `detail` string [TypeUtils.java:isEquivalentType].

## Side Effects

- `TypeHandleASM.of` reads class files through the `Filer`. Its results are
  kept in a static cache that is never cleared for the life of the class
  loader, so they carry over between compilations [TypeHandleASM.java:cache].
- `TypeReference.getHandle` prints a stack trace when resolution fails and
  then returns `null` [TypeReference.java:getHandle].

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileApJava`
- Run: `mise exec -- ./gradlew test`. Note that `src/test` does not exist yet,
  so no tests cover this package today.
