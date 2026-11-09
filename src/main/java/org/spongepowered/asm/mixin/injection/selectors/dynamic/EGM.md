# selectors/dynamic

## Overview

The `@Desc` dynamic target selector. A `DynamicSelectorDesc` matches members by
owner, name, argument types and return type, using values from a `@Desc`
annotation. The annotation can be local to the consumer, given by id, or found
from implicit coordinates. `DescriptorResolver` finds the annotation by
searching outward from the consuming annotation, through the handler method,
the mixin class, and each parent selector context. The annotations are
declared in src/main/java/org/spongepowered/asm/mixin/injection/Desc.java and
Descriptors.java.

## Interface

- `public interface IResolvedDescriptor`. Its methods are `isResolved()`,
  `isDebug()`, `getAnnotation()`, `getResolutionInfo()`, `getId()`,
  `getOwner()`, `getName()`, `getArgs()`, `getReturnType()`, `getMatches()`
  (returns a `Quantifier`) and `getNext()` [IResolvedDescriptor.java:IResolvedDescriptor].
- `public final class DescriptorResolver` has a private constructor
  [DescriptorResolver.java:DescriptorResolver].
  - `public static String PRINT_ID = "?"`. This field is mutable, not final
    [DescriptorResolver.java:PRINT_ID].
  - `static IResolvedDescriptor resolve(IAnnotationHandle desc, ISelectorContext context)`
    wraps an annotation the caller already has. It does no search and records
    no searched coordinates [DescriptorResolver.java:resolve(IAnnotationHandle,ISelectorContext)].
  - `static IResolvedDescriptor resolve(String id, ISelectorContext context)`
    searches by id. An empty id searches by implicit coordinate
    [DescriptorResolver.java:resolve(String,ISelectorContext)].
- `public class DynamicSelectorDesc implements ITargetSelectorDynamic, ITargetSelectorByName`,
  annotated `@SelectorId("Desc")` and `@SelectorAnnotation(Desc.class)`
  [DynamicSelectorDesc.java:DynamicSelectorDesc].
  - Factories: `parse(String input, ISelectorContext)`,
    `parse(IAnnotationHandle, ISelectorContext)`, `resolve(ISelectorContext)`,
    `of(IAnnotationHandle, ISelectorContext)` and `of(IResolvedDescriptor)`
    [DynamicSelectorDesc.java:parse; resolve; of].
  - Accessors: `getId()`, `getOwner()` (internal name), `getName()`,
    `getArgs()`, `getReturnType()`, `getDesc()` (the method descriptor) and
    `toDescriptor()` [DynamicSelectorDesc.java:getOwner; getDesc; toDescriptor].
  - Selector methods: `validate()`, `next()`, `configure(Configure, String...)`,
    `attach(ISelectorContext)`, `getMinMatchCount()`, `getMaxMatchCount()`,
    `matches(String, String, String)` and `match(ElementNode)`
    [DynamicSelectorDesc.java:validate; configure; match].

## Invariants

- The resolver searches in this order:
  1. the `desc` values on the current selector annotation;
  2. `@Descriptors` and then `@Desc` on the handler method;
  3. `@Descriptors` and then `@Desc` on the mixin, at the coordinate prefixed
     with the root context coordinate;
  4. the parent context, with `parent.getSelectorCoordinate(false) + "." + coordinate`.

  It returns `null` when the root is reached with no match
  [DescriptorResolver.java:resolve(String,ISelectorContext,IResolverObserver,String)].
- A candidate `@Desc` matches when its `id` equals the requested id, ignoring
  case. When the id is empty (implicit), its `id` must equal the current
  coordinate, ignoring case. A missing `id` defaults to the coordinate being
  visited [DescriptorResolver.java:resolve(...,IAnnotationHandle)].
- The id `"?"` (`PRINT_ID`) switches to the debug observer. The search then
  runs with an empty id and the result has `isDebug() == true`
  [DescriptorResolver.java:resolve(String,ISelectorContext)].
- Reading annotations from an element that is not a `MethodNode`, `ClassNode`,
  `MixinTargetContext`, `IAnnotatedElement` or `null` throws
  `IllegalStateException("Cannot read visible annotations from element with unknown type: " + className)`
  [DescriptorResolver.java:getVisibleAnnotation].
- An unresolved descriptor returns `""` for its id and name, `Type.VOID_TYPE`
  for its owner and return type, an empty argument array, `Quantifier.DEFAULT`
  for matches and an empty `next` list
  [DescriptorResolver.java:Descriptor].
- In a resolved descriptor:
  - the name is taken from `value` when that is non-empty, otherwise from
    `name`;
  - an owner of `void` is replaced by the mixin's target class;
  - `getMatches()` uses `min` (clamped to at least 0) and `max`. A `max` of 0
    or less means `Integer.MAX_VALUE`, and a missing `max` gives -1
  [DescriptorResolver.java:Descriptor.getName; getOwner; getMatches].
- `getResolutionInfo()` returns `""` when nothing was searched. Otherwise it
  returns `Searched coordinates [ "a", "b" ]`
  [DescriptorResolver.java:Descriptor.getResolutionInfo].
- `DynamicSelectorDesc.parse(String, ...)` does not throw when resolution fails
  without debug. It returns a disabled selector that holds
  `InvalidSelectorException("Could not resolve @Desc(" + input + ") for " + context + extra)`.
  `extra` is the resolution info and is added only when `input` is empty
  [DynamicSelectorDesc.java:parse(String,ISelectorContext)].
- `parse(IAnnotationHandle, ...)` handles failure the same way, with the
  message `"Invalid descriptor"`. `resolve(ISelectorContext)` and
  `of(IAnnotationHandle, ...)` return `null` instead
  [DynamicSelectorDesc.java:parse(IAnnotationHandle,ISelectorContext); resolve; of].
- `validate()` throws the stored parse exception when there is one and
  otherwise returns `this` [DynamicSelectorDesc.java:validate].
- A disabled selector or a `null` node always gives `MatchResult.NONE`
  [DynamicSelectorDesc.java:match].
- Matching rules:
  - Fields are compared with the descriptor of `returnType`. Methods are
    compared with `Bytecode.getDescriptor(returnType, args)`. A different
    descriptor gives `NONE`.
  - When the owner is not `void` it must equal the candidate's owner.
  - The name gives `EXACT_MATCH` when it is equal, `MATCH` when it is equal
    ignoring case, and `NONE` otherwise.
  - A `null` name gives `EXACT_MATCH` for any candidate name.
  [DynamicSelectorDesc.java:matches(String,String,String,String)]
- `configure` returns a new selector in these cases and `this` otherwise:
  - `SELECT_MEMBER` with default matches becomes `Quantifier.SINGLE`.
  - `SELECT_INSTRUCTION` with default matches becomes `Quantifier.ANY`.
  - `MOVE` replaces the owner with `args[0]`.
  - `CLEAR_LIMITS` becomes `ANY` when the limits are not already 0 to
    `Integer.MAX_VALUE`.

  It calls `request.checkArgs(args)` before anything else
  [DynamicSelectorDesc.java:configure].
- `next()` walks the `next` annotations by index. Each one is resolved with a
  `null` context, so a `void` owner stays `void` in those selectors. It
  returns `null` past the end [DynamicSelectorDesc.java:next(int); Next].
- `attach` returns `this` unchanged [DynamicSelectorDesc.java:attach].

## Side Effects

- When the id is `"?"`, `ResolverObserverDebug` prints a table of the visited
  coordinates through `PrettyPrinter.print()` after resolution
  [DescriptorResolver.java:ResolverObserverDebug.postResolve].
- None beyond that. The resolver keeps no static cache.

## Verification

- Compile: `mise exec -- ./gradlew compileJava` (run from the repository root).
- Test: `mise exec -- ./gradlew test`. `build.gradle` declares the `test`
  source set, but `src/test` does not exist yet, so there are no tests today.
