# org.spongepowered.asm.mixin.gen

## Overview

This package holds the `@Accessor` and `@Invoker` annotations and the runtime code that turns an abstract annotated mixin method into a synthetic method body. The pipeline has three steps:

1. `AccessorInfo` reads the annotated method and works out an `AccessorType`, the target name and a target selector.
2. `locate()` finds the target field or method in the target `ClassNode`.
3. `validate()` and `generate()` pick an `AccessorGenerator` subclass and emit the bytecode.

The pipeline is driven by src/main/java/org/spongepowered/asm/mixin/transformer/MixinTargetContext.java. It calls `AccessorInfo.of(...)` when it registers the method, then calls `accessor.locate()`, `accessor.validate()` and `accessor.generate()` in that order.

## Interface

- `@Accessor`: `@Target(METHOD)`, `@Retention(RUNTIME)`, `String value() default ""` and `boolean remap() default true` [Accessor.java:Accessor]
- `@Invoker`: `@Target(METHOD)`, `@Retention(RUNTIME)`, `String value() default ""` and `boolean remap() default true`. The javadoc says `<init>` or the fully-qualified target class name selects a constructor [Invoker.java:Invoker]
- `public class AccessorInfo extends SpecialMethodInfo` [AccessorInfo.java:AccessorInfo]. `SpecialMethodInfo` is in the sibling package `mixin/struct`.
  - `public AccessorInfo(MixinTargetContext mixin, MethodNode method)` [AccessorInfo.java:AccessorInfo]
  - `public static AccessorInfo of(MixinTargetContext mixin, MethodNode method, Class<? extends Annotation> type)` [AccessorInfo.java:of]
  - `void locate()`, `void validate()` and `MethodNode generate()` [AccessorInfo.java:locate] [AccessorInfo.java:validate] [AccessorInfo.java:generate]
  - Getters: `ITargetSelector getTarget()`, `Type getTargetFieldType()`, `FieldNode getTargetField()`, `MethodNode getTargetMethod()`, `Type getReturnType()`, `Type[] getArgTypes()` and `boolean isStatic()` [AccessorInfo.java:getTarget]
  - `public static String inflectTarget(String name, AccessorType type, String description, ISelectorContext context, boolean verbose)` and an overload that takes an `AccessorName` [AccessorInfo.java:inflectTarget]. The annotation processor in src/ap also calls this method.
  - `toString()` returns `<mixin>->@<Annotation>[<type or UNPARSED_ACCESSOR>]::<methodName><desc>` [AccessorInfo.java:toString]
- `public enum AccessorInfo.AccessorType` [AccessorInfo.java:AccessorType]:
  - `FIELD_GETTER` takes the prefixes `get` and `is`.
  - `FIELD_SETTER` takes `set`.
  - `METHOD_PROXY` takes `call` and `invoke`.
  - `OBJECT_FACTORY` takes `new` and `create`.
  - Methods: `boolean isExpectedPrefix(String)`, `Set<String> getExpectedPrefixes()` (unmodifiable), and the package-private `AccessorGenerator getGenerator(AccessorInfo)`.
- `public static final class AccessorInfo.AccessorName` has the public final fields `methodName`, `prefix` and `name`, plus `static AccessorName of(String methodName)` and `of(String methodName, boolean toMemberCase)` [AccessorInfo.java:AccessorName]
- `class InvokerInfo extends AccessorInfo` is package-private [InvokerInfo.java:InvokerInfo]
- `public abstract class AccessorGenerator` [AccessorGenerator.java:AccessorGenerator]:
  - constructor `(AccessorInfo info, boolean isStatic)`
  - `void validate()` (empty by default) and `abstract MethodNode generate()`
  - protected `checkModifiers()` and `createMethod(int maxLocals, int maxStack)`
- Field generators: `AccessorGeneratorField` (abstract), `AccessorGeneratorFieldGetter` and `AccessorGeneratorFieldSetter` [AccessorGeneratorField.java:AccessorGeneratorField] [AccessorGeneratorFieldGetter.java:generate] [AccessorGeneratorFieldSetter.java:generate]
- Method generators: `AccessorGeneratorMethodProxy` and `AccessorGeneratorObjectFactory extends AccessorGeneratorMethodProxy` [AccessorGeneratorMethodProxy.java:generate] [AccessorGeneratorObjectFactory.java:generate]
- Subpackage: [throwables](./throwables/EGM.md) holds `InvalidAccessorException`.

## Invariants

### Name inflection

- `AccessorName.PATTERN` is `^(get|is|set|call|invoke|new|create)(([A-Z])(.*?))(_\$md.*)?$`. The prefix list is built from `AccessorType.values()` in declaration order. A prefix must be followed by an upper-case letter, and a trailing `_$md...` suffix is dropped from the name. `of` returns null when the name does not match [AccessorInfo.java:AccessorName.of] [AccessorInfo.java:getPrefixList].
- `AccessorName.of(name, toMemberCase)` lower-cases the first character (`Locale.ROOT`) only when `toMemberCase` is true and the name part is not entirely upper case. So `getFoo` gives `foo`, and `getFOO_BAR` keeps `FOO_BAR`. `of(String)` passes `toMemberCase = true` [AccessorInfo.java:AccessorName.of].
- A prefix that does not match the computed type is not an error. When `verbose` is true, Mixin logs the warning `Unexpected prefix for {}, found [{}] expecting {}` [AccessorInfo.java:inflectTarget]. `AccessorInfo.inflectTarget()` passes `Option.DEBUG_VERBOSE` as the `verbose` flag.

### @Accessor resolution

- The type comes from the descriptor, not from the name. A `void` return makes the accessor `FIELD_SETTER`; any other return type makes it `FIELD_GETTER` [AccessorInfo.java:initType].
- A getter must take 0 arguments and a setter exactly 1. Otherwise Mixin throws `InvalidAccessorException` with the message `<this> must take exactly 0 arguments, found N` or `<this> must take exactly 1 argument, found N`. The target field type is the getter's return type or the setter's single argument type [AccessorInfo.java:initTargetFieldType].
- When `value` is null or empty, the target name is inflected from the method name. If inflection fails, Mixin throws `InvalidAccessorException("Failed to inflect target name for <this>, supported prefixes: <prefixes>")`. Otherwise the name goes through `TargetSelector.parseName` [AccessorInfo.java:getTargetName].
- The target selector is `new MemberInfo(name, null, targetFieldType.getDescriptor())`, which matches by name and field descriptor with any owner [AccessorInfo.java:initTarget].
- `locate()` runs `TargetSelector.run(target.configure(Configure.ORPHAN), ...)` and then `getSingleResult(true)`. An `IllegalStateException` from no match or several matches is rethrown as `InvalidAccessorException("<msg> matching <target> in <class> for <this>")` [AccessorInfo.java:findTarget].
- `AccessorInfo.of` returns an `AccessorInfo` for `Accessor.class` and an `InvokerInfo` for `Invoker.class`. Any other annotation type throws `InvalidAccessorException("Could not parse accessor for unknown type <name>")` [AccessorInfo.java:of].

### @Invoker resolution

- When `value` is set, it is first remapped with `mixin.getReferenceMapper().remap(mixin.getClassRef(), value)` and `.` is replaced with `/`. The invoker is `OBJECT_FACTORY` if the result equals `<init>` or the target class ref [InvokerInfo.java:initType].
- When `value` is absent, the method name is parsed with `AccessorName.of(name, false)`. If the prefix is `new` or `create` and the name part equals the target class's simple name, the invoker is `OBJECT_FACTORY`. Every other case is `METHOD_PROXY` [InvokerInfo.java:initType].
- An `OBJECT_FACTORY` invoker must return exactly the target class type, or Mixin throws `InvalidAccessorException("... appears to have an invalid return type ...")`. It must also be static, or Mixin throws `InvalidAccessorException("<this> for OBJECT_FACTORY must be static")` [InvokerInfo.java:initType].
- The selector for `OBJECT_FACTORY` is `MemberInfo("<init>", null, <invoker desc with return type V>)`. The selector for `METHOD_PROXY` is `MemberInfo(name, null, <invoker desc>)`, so the invoker descriptor must match the target method exactly [InvokerInfo.java:initTarget].
- `getTargetFieldType()` is null for invokers [InvokerInfo.java:initTargetFieldType].
- When a `METHOD_PROXY` lookup fails and `value` contained `.` or `/`, the message starts with `Invalid factory invoker failed to match the target class. ` [InvokerInfo.java:findTargetMethod].

### Generators

- `validate()` creates the generator with `type.getGenerator(this)` and then calls `generator.validate()`. Checks made in generator constructors therefore fire during `validate()` [AccessorInfo.java:validate].
- `checkModifiers` rejects a static accessor whose target is not static. It throws `InvalidInjectionException("<info> is invalid. Accessor method is static but the target is not.")`, from src/main/java/org/spongepowered/asm/mixin/injection/throwables. An instance accessor for a static target is allowed. Field generators call this check, and so does the public `AccessorGeneratorMethodProxy` constructor [AccessorGenerator.java:checkModifiers] [AccessorGeneratorField.java:AccessorGeneratorField] [AccessorGeneratorMethodProxy.java:AccessorGeneratorMethodProxy].
- `AccessorGeneratorObjectFactory` uses the protected proxy constructor, which skips `checkModifiers`. A non-static accessor throws `InvalidInjectionException("<info> is invalid. Factory method must be static.")` [AccessorGeneratorObjectFactory.java:AccessorGeneratorObjectFactory].
- A generated method copies the abstract method's name and descriptor. Its access flags are `(access & ~ACC_ABSTRACT) | ACC_SYNTHETIC`, and `visibleAnnotations` contains only the accessor annotation node [AccessorGenerator.java:createMethod]. `generate()` then merges the original method's annotations into it [AccessorInfo.java:generate].
- Getter bytecode is `ALOAD 0` (instance targets only), then `GETFIELD` or `GETSTATIC` on the target class, then the `xRETURN` for the field type [AccessorGeneratorFieldGetter.java:generate].
- Setter bytecode is `ALOAD 0` (instance targets only), then `xLOAD` from slot 0 (static) or 1 (instance), then `PUTFIELD` or `PUTSTATIC`, then `RETURN` [AccessorGeneratorFieldSetter.java:generate].
- A setter for a final field without `@Mutable` is still generated. When `DEBUG_VERBOSE` is on, Mixin logs `{} for final field {}::{} is not @Mutable` [AccessorGeneratorFieldSetter.java:validate].
- Method proxy opcode selection [AccessorGeneratorMethodProxy.java:generate]:
  - A static target uses `INVOKESTATIC`.
  - An interface target uses `INVOKESPECIAL` when the method is both private and synthetic, and `INVOKEINTERFACE` otherwise.
  - A class target uses `INVOKESPECIAL` when the method is private, and `INVOKEVIRTUAL` otherwise.
  - Arguments load from slot 0 for a static accessor and slot 1 otherwise. The `itf` flag equals `targetIsInterface`.
- Object factory bytecode is `NEW <target>`, then `DUP` (or `DUP2` when the return size is 2), then the arguments from slot 0, then `INVOKESPECIAL <init>` with the target constructor's descriptor, then `ARETURN` [AccessorGeneratorObjectFactory.java:generate].

## Side Effects

- The `AccessorInfo` constructor writes a `target` value into the accessor's `AnnotationNode` with `annotation.visit("target", target.toString())` [AccessorInfo.java:AccessorInfo].
- When the setter method is `@Mutable` (`ClassInfo.Method.isDecoratedMutable()`), `AccessorGeneratorFieldSetter.generate()` clears `ACC_FINAL` on the target `FieldNode` in the target class [AccessorGeneratorFieldSetter.java:generate].
- Warning logs go through `MixinService.getService().getLogger("mixin")`, and only when `DEBUG_VERBOSE` is on [AccessorInfo.java:inflectTarget] [AccessorGeneratorFieldSetter.java:validate].
- The package has no mutable static state. The only static field is the compiled `AccessorName.PATTERN`, built when the class initializes [AccessorInfo.java:AccessorName].

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileJava`
- Run from the repo root: `mise exec -- ./gradlew test`
- `build.gradle` declares a `test` source set, but `src/test` does not exist yet. `test` therefore runs no tests today.
