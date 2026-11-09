# code

## Overview

The injector base class and its instruction-list views. `Injector` runs the
find, preInject, inject and postInject cycle for one handler method. It also
validates the handler signature, including `@Coerce` rules, and emits the
handler call. `InjectorTarget` wraps a `Target` for one injector and caches a
read-only instruction view for each slice. `MethodSlice` and `MethodSlices`
turn `@Slice` annotations into instruction ranges. `InsnListReadOnly` and
`InsnListEx` are the read-only `InsnList` views that injection points search.

## Interface

- `public abstract class Injector` [Injector.java:Injector]
  - `Injector(InjectionInfo info, String annotationType)`.
  - `final List<InjectionNode> find(InjectorTarget, List<InjectionPoint>)`.
  - `final void preInject(Target, List<InjectionNode>)` and
    `final void inject(Target, List<InjectionNode>)`.
  - Protected hooks: `addTargetNode`, `findTargetNodes`, `sanityCheck`,
    `checkTargetModifiers`, `checkTargetForNode`, `preInject`,
    `abstract inject`, `postInject`, `invokeHandler`, `invokeHandlerWithArgs`,
    `storeArgs`, `pushArgs`, `validateParams`, `checkCoerce` and
    `throwException`.
  - `public static boolean canCoerce(Type, Type)`, `canCoerce(String, String)`
    and `canCoerce(char, char)`.
  - Nested `public static final class TargetNode` with `getNode()` and
    `getNominators()`.
  - Nested `public static class InjectorData` with the fields `target`,
    `description` (default `"handler"`), `allowCoerceArgs` (default `true`),
    `captureTargetArgs` (default 0) and `coerceReturnType` (default `false`).
- `public class InjectorTarget` [InjectorTarget.java:InjectorTarget]
  - `InjectorTarget(ISliceContext, Target, SelectedMethod)`.
  - `addInjectionNode` and `getInjectionNode`.
  - `getName`, `getDesc`, `getSignature`, `getTarget`, `getMethod` and
    `getSelectedMethod`.
  - `isMerged`, `getMergedBy` and `getMergedPriority`.
  - `getSlice(String id)`, `getSlice(InjectionPoint)` and `dispose()`.
- `public interface ISliceContext extends IInjectionPointContext` declares
  `MethodSlice getSlice(String id)` [ISliceContext.java:ISliceContext].
- `public interface IInsnListEx` [IInsnListEx.java:IInsnListEx]
  - Target metadata: `getTargetName`, `getTargetDesc`, `getTargetSignature`,
    `getTargetAccess`, `isTargetStatic`, `isTargetConstructor` and
    `isTargetStaticInitialiser`.
  - `getSpecialNode(SpecialNodeType)`, where `SpecialNodeType` is
    `DELEGATE_CTOR`, `INITIALISER_INJECTION_POINT` or `CTOR_BODY`.
  - Decoration readers: `hasDecoration`, `getDecoration(key)` and
    `getDecoration(key, default)`.
- `public abstract class InsnListReadOnly extends InsnList` (the ASM class)
  [InsnListReadOnly.java:InsnListReadOnly].
- `public class InsnListEx extends InsnListReadOnly implements IInsnListEx`.
  Beyond the interface it adds `decorate(key, value)`, `undecorate(key)` and
  `undecorate()` [InsnListEx.java:InsnListEx].
- `public final class MethodSlice` [MethodSlice.java:MethodSlice]
  - `static parse(ISliceContext, Slice)` and
    `static parse(ISliceContext, AnnotationNode)`.
  - `getId()`, `InsnListReadOnly getSlice(Target)` and `postInject()`.
- `public final class MethodSlices` provides `static parse(InjectionInfo)`,
  `get(String id)` and `postInject()` [MethodSlices.java:MethodSlices].

## Invariants

- `find` calls `sanityCheck` first. It then collects nodes from every injection
  point into a `TreeMap` keyed by instruction index, so the target nodes it
  returns are in instruction order with no duplicates. Each `TargetNode`
  records the set of injection points that nominated it
  [Injector.java:find; findTargetNodes; addTargetNode(InjectorTarget,Map,...)].
- Merged targets: a target carries `@MixinMerged` from another mixin. For
  that target, `injectionPoint.checkPriority(mergedPriority, mixinPriority)`
  must return true. Otherwise the injector throws
  `InvalidInjectionException("%s on %s with priority %d cannot inject into %s merged by %s with priority %d")`.
  When the annotation has no priority, `mergedPriority` defaults to
  `IMixinConfig.DEFAULT_PRIORITY` [Injector.java:findTargetNodes; InjectorTarget.java:InjectorTarget].
- Injection point specifiers (the default is `ALL`):
  - `:ONE` must match exactly one instruction. Any other count throws
    `InvalidInjectionException("%s on %s has specifier :ONE but matched %d instructions")`.
  - `:FIRST` and `:LAST` keep only the first or the last match.
  [Injector.java:findTargetNodes]
- `sanityCheck` throws
  `InvalidInjectionException("Target class does not match injector class in ...")`
  when `target.classNode` is not the injector's class node [Injector.java:sanityCheck].
- `checkTargetModifiers`:
  - With `exactMatch`, the handler and the target must both be static or both
    be instance methods. Otherwise it throws
    `"'static' modifier of handler method does not match target in %s"`.
  - Without it, only a non-static handler on a static target is rejected, with
    `"non-static callback method %s targets a static method which is not supported"`.
  [Injector.java:checkTargetModifiers]
- `checkTargetForNode` on a `Constructor` target:
  - `METHODS_ONLY` throws `"Found %s targetting a constructor in injector %s"`.
  - A missing delegate initialiser throws `InjectionError("Delegate constructor lookup failed ...")`.
  - A node at or before the `super()`/`this()` call throws under
    `CONSTRUCTORS_AFTER_DELEGATE`. Otherwise such a node must have a static
    handler, or it throws `"%s handler before %s() invocation must be static ..."`.
  - Nodes after the delegate call fall through to
    `checkTargetModifiers(target, false)`.
  [Injector.java:checkTargetForNode]
- `inject(Target, List)` skips nodes whose `isRemoved()` is true. It logs a
  WARN for each skip, but only when `DEBUG_VERBOSE` is on. It runs every
  `inject` before any `postInject` [Injector.java:inject(Target,List)].
- The handler call opcode depends on the handler:
  - `INVOKESTATIC` for a static handler;
  - for an interface, `INVOKESPECIAL` when the handler is private and
    synthetic, otherwise `INVOKEINTERFACE`;
  - otherwise `INVOKESPECIAL` for a private handler and `INVOKEVIRTUAL` for
    the rest.

  Each emitted call records `info.addCallbackInvocation(handler)`
  [Injector.java:invokeHandler(InsnList,MethodNode)].
- `validateParams` checks the return type and then each expected argument.
  Remaining handler parameters are matched in order against the target
  method's arguments and counted in `captureTargetArgs`.
  - Any handler parameters still left over throw
    `"... Found %d unexpected additional method arguments: ..."`.
  - Validation errors are rethrown as `InvalidInjectionException` with the
    suffix `". Handler signature: <desc> Expected signature: <desc>"`.
  [Injector.java:validateParams]
- `checkCoerce` rules:
  - An index beyond the handler's arguments throws `"Not enough arguments ..."`.
  - Identical types pass. A redundant `@Coerce` is logged at INFO only under
    `DEBUG_VERBOSE`.
  - Different types without `@Coerce`, or where coercion is not allowed, throw
    `"Found unexpected %s type ..."`. The message ends with
    `". @Coerce not allowed here"` when `@Coerce` was present.
  - When coercion is impossible it throws `"Cannot @Coerce ..."`.
  [Injector.java:checkCoerce]
- `canCoerce(from, to)`. In `checkCoerce`, `from` is the type the handler
  declares and `to` is the type the injector expects.
  - For primitive descriptors (one character), it is true only when `to` is
    `I` and `from` is one of `I`, `B`, `S`, `C` or `Z`.
  - For object types, it is true when `to` is the same `ClassInfo` as `from`
    or has `from` as a superclass. The handler may therefore declare a
    supertype of the expected type.
  - Arrays must have equal dimensions, and their element types must pass the
    object rule.
  - Any other combination is false.
  [Injector.java:canCoerce]
- `storeArgs` stores arguments in reverse order and adds a local named
  `"injectorAllocatedLocal%d"` for each one [Injector.java:storeArgs].
- Every mutating method of `InsnListReadOnly` (`set`, `add`, `insert`,
  `insertBefore`, `remove`) is `final` and throws `UnsupportedOperationException`.
  `contains(null)` returns `false` [InsnListReadOnly.java:add; contains].
- `InsnListEx.getSpecialNode` returns `null` unless the target is a
  `Constructor` and the node lies inside this list. `INITIALISER_INJECTION_POINT`
  always uses `Initialiser.InjectionMode.DEFAULT` [InsnListEx.java:getSpecialNode].
- `InsnListEx.decorate` calls `value.replace(previous)` when an
  `IChainedDecoration` replaces a previous value of the same class.
  `hasDecoration` is false for keys whose value is `null`
  [InsnListEx.java:decorate; hasDecoration].
- `isTargetStaticInitialiser()` is true when the target name equals
  `Constants.CLINIT` [InsnListEx.java:isTargetStaticInitialiser].
- `MethodSlice.getSlice` resolution:
  - `from` defaults to index 0 and becomes 0 when it does not match.
  - `to` defaults to the last index and becomes `start` when it does not
    match.
  - The full range gives a plain `InsnListEx`. Any other range gives an
    `InsnListSlice` limited to `[start, end]`.
  [MethodSlice.java:getSlice; find]
- `MethodSlice.getSlice` throws `InvalidSliceException("%s is negative size. Range(%d -> %d)")`
  when start is after end. Out-of-bounds indices throw
  `InjectionError("Unexpected critical error in ...")` [MethodSlice.java:getSlice].
- Slice injection points use `FIRST` as the default specifier.
  - `ALL` throws `InvalidSliceException("ALL is not a valid specifier for slice ...")`.
  - `ONE` with a match count other than 1 throws `"%s requires 1 result but found %d"`.
  - `LAST` takes the last node.
  [MethodSlice.java:find]
- When both `from` and `to` are `null`, the constructor tries to throw
  `InvalidSliceException("%s is redundant. No 'from' or 'to' value specified")`.
  The message formats `this` before `owner` and `name` are assigned, and
  `describeSlice` then dereferences the still-`null` owner
  [MethodSlice.java:MethodSlice; describeSlice].
- Slice names are `"@Slice[" + id + "]"`. The slice context coordinate is
  `"slice"`, or `"slice." + id` when an id is set
  [MethodSlice.java:getSliceName; parse(ISliceContext,AnnotationNode)].
- The slice iterator rejects `remove`, `set` and `add` with
  `UnsupportedOperationException`. `next()` past the end and `previous()`
  before the start throw `NoSuchElementException`. `indexOf` returns -1 outside
  the slice [MethodSlice.java:InsnListSlice.SliceIterator; InsnListSlice.indexOf].
- `MethodSlices.add` uses `info.getSliceId(slice.getId())` as the key. A second
  slice with the same key throws
  `InvalidSliceException("<slice> has a duplicate id, '<id>' was already defined")`
  [MethodSlices.java:add].
- `InjectorTarget.getSlice(id)` caches one view per id. An id with no
  `MethodSlice` gives a full-method `InsnListEx` [InjectorTarget.java:getSlice].

## Side Effects

- `Injector` and `MethodSlice` each get a static logger from
  `MixinService.getService().getLogger("mixin")`
  [Injector.java:logger; MethodSlice.java:logger].
- Injectors modify `Target` instruction lists, local variable tables, and max
  stack and max locals through `storeArgs`, `invokeHandler` and
  `throwException`. `throwException` adds 3 to the stack extension
  [Injector.java:storeArgs; throwException].
- Under `DEBUG_VERBOSE`, `MethodSlice.postInject` logs a WARN
  `"{} did not match any instructions"` for a `from` or `to` that never matched
  [MethodSlice.java:postInject].
- `InjectorTarget.dispose()` disposes every cached read-only view and clears the
  cache. Disposed views drop their backing list, so they must not be used
  afterwards [InjectorTarget.java:dispose; InsnListReadOnly.java:dispose].

## Verification

- Compile: `mise exec -- ./gradlew compileJava` (run from the repository root).
- Test: `mise exec -- ./gradlew test`. `build.gradle` declares the `test`
  source set, but `src/test` does not exist yet, so there are no tests today.
