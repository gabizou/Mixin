# selectors

## Overview

The target selector system. It turns the `method`, `target` and `@At.target`
values of injector annotations into `ITargetSelector`s, and matches those
selectors against methods, fields and instructions. A selector string takes
one of three forms:

- an explicit member, parsed into `MemberInfo` (see
  src/main/java/org/spongepowered/asm/mixin/injection/struct/MemberInfo.java);
- a regex form, parsed into `MemberMatcher`;
- an `@Id(...)` dynamic selector, looked up in a registry. The built-in one is
  `@Desc`.

`TargetSelectors` runs the selectors against a target class to choose the
methods an injector will target.

## Interface

- `public interface ITargetSelector` [ITargetSelector.java:ITargetSelector]
  - Methods: `next()`, `configure(Configure, String...)`, `validate()`,
    `attach(ISelectorContext)`, `getMinMatchCount()`, `getMaxMatchCount()` and
    `<TNode> MatchResult match(ElementNode<TNode>)`.
  - `enum Configure` lists each request with the number of arguments it
    needs: `SELECT_MEMBER(0)`, `SELECT_INSTRUCTION(0)`, `MOVE(1)`, `ORPHAN(0)`,
    `TRANSFORM(1)`, `PERMISSIVE(0)` and `CLEAR_LIMITS(0)`. It provides
    `checkArgs(String...)`.
- Sub-interfaces:
  - `ITargetSelectorByName` adds `getOwner`, `getName`, `getDesc`,
    `toDescriptor` and `matches(owner, name, desc)`.
  - `ITargetSelectorConstructor` adds `toCtorType` and `toCtorDesc`.
  - `ITargetSelectorRemappable` adds `isFullyQualified`, `isField`,
    `isConstructor`, `isClassInitialiser`, `isInitialiser`, `asMapping`,
    `asMethodMapping`, `asFieldMapping`, `move`, `transform` and `remapUsing`.
  - `ITargetSelectorDynamic` declares the runtime annotations
    `@SelectorId(value, namespace = "")` and
    `@SelectorAnnotation(Class<? extends Annotation>)`.
  [ITargetSelectorByName.java; ITargetSelectorConstructor.java; ITargetSelectorRemappable.java; ITargetSelectorDynamic.java]
- `public interface ISelectorContext` declares `getParent`, `getMixin`,
  `getMethod`, `getAnnotation`, `getSelectorAnnotation`,
  `getSelectorCoordinate(boolean leaf)`, `remap` and `getElementDescription`
  [ISelectorContext.java:ISelectorContext].
- `public enum MatchResult` has the constants `NONE`, `WEAK`, `MATCH` and
  `EXACT_MATCH`, and the methods `isAtLeast`, `isMatch` and `isExactMatch`
  [MatchResult.java:MatchResult].
- `public final class TargetSelector` is a static utility with a private
  constructor [TargetSelector.java:TargetSelector]
  - Registration: `register(Class<? extends ITargetSelectorDynamic>, String namespace)`.
  - Parsing: `parse(String | IAnnotationHandle, ISelectorContext)`,
    `parse(Iterable<?>, ISelectorContext[, Set])`, `parseAndValidate(...)`
    (same overloads) and `parseName(String, ISelectorContext)`.
  - Running: `run(ITargetSelector | Iterable<ITargetSelector>, Iterable<ElementNode<TNode>>)`
    returns a `Result<TNode>` with the public fields `exactMatch` and
    `candidates`, and the method `getSingleResult(boolean strict)`.
- `public class TargetSelectors implements Iterable<SelectedMethod>`
  [TargetSelectors.java:TargetSelectors]
  - `TargetSelectors(ISelectorContext, ClassNode)`.
  - Building: `parse(Set<ITargetSelector>)` and `addSelector`.
  - Running: `find()` and `validate(int expected, int required)`.
  - Access: `size`, `clear`, `remove`, `iterator`, `isPermissivePassEnabled`
    and `setPermissivePass`.
  - `SelectedMethod` provides `getParent`, `next` and `getMethod`.
- `public final class MemberMatcher implements ITargetSelector` provides
  `static parse(String, ISelectorContext)` [MemberMatcher.java:MemberMatcher].
- `public abstract class ElementNode<TNode>` [ElementNode.java:ElementNode]
  - Accessors: `getType()` (returns a `NodeType`), `isField`, `getMethod`,
    `getField`, `getInsn`, `getOwner`, `getName`, `getSyntheticName`,
    `getDesc`, `getDelegateDesc`, `getImplDesc`, `getSignature` and `get`.
  - Factories: `of(ClassNode, MethodNode | FieldNode | TNode)`,
    `of(AbstractInsnNode)`, `listOf`, `fieldList`, `methodList`, `insnList`
    and `dynamicInsnList`.
  - `NodeType` has the constants `UNDEFINED`, `METHOD`, `FIELD`, `METHOD_INSN`,
    `FIELD_INSN` and `INVOKEDYNAMIC_INSN`.
- `public class InvalidSelector implements ITargetSelector` is the placeholder
  for a selector that failed to parse [InvalidSelector.java:InvalidSelector].
- `public class InvalidSelectorException extends MixinException`. It is
  unchecked [InvalidSelectorException.java:InvalidSelectorException].
- Subdirectories:
  - [dynamic](./dynamic/EGM.md): the `@Desc` dynamic selector
    (`DynamicSelectorDesc`) and `DescriptorResolver`, which resolves `@Desc` by
    coordinate or id.
  - [throwables](./throwables/EGM.md): `SelectorException` and
    `SelectorConstraintException`.

## Invariants

- `TargetSelector.parse(String)` trims the input, then decides as follows:
  - Input ending in `/` is parsed as a `MemberMatcher`.
  - Input not starting with `@` is parsed with `MemberInfo.parse`.
  - Otherwise it must match `(?i)^@([a-z]+(:[a-z]+)?)(\((.*)\))?$`, and the id
    is looked up in lower case.
  [TargetSelector.java:parse(String,ISelectorContext)]
- Parse errors are not thrown. Each one returns an `InvalidSelector` that
  throws later, from `validate()` or `match()`:
  - an unrecognised `@` format: `"Dynamic selector was in an unrecognised format. Parsing selector: ..."`;
  - an unknown id: `"Dynamic selector with id '@...' is not registered. Parsing selector: ..."`;
  - an annotation with no registered selector: `"Dynamic selector for annotation ... is not registered."`;
  - an exception from a parse method, which becomes the cause.
  [TargetSelector.java:parse(String,ISelectorContext); parse(IAnnotationHandle,ISelectorContext)]
- `InvalidSelector.validate()` rethrows a cause that is an
  `InvalidSelectorException`. Otherwise it throws
  `InvalidSelectorException("Error parsing target selector[, the input was in an unexpected format: <input>]")`.
  `match()` calls `validate()`, and both match counts are 0
  [InvalidSelector.java:validate; match].
- The dynamic selector registry is a static `LinkedHashMap`. The built-in
  `DynamicSelectorDesc` is registered under `desc` and `mixin:desc`.
  `register` rules:
  - It needs `@SelectorId`, or it throws `IllegalArgumentException`.
  - It needs a namespace, from the annotation or the argument, or it throws
    `IllegalArgumentException("... has no namespace ...")`.
  - The code `namespace:id` is lower-cased and must match
    `[a-z]+(:[a-z]+)?`, or it throws `IllegalArgumentException("... has an invalid id ...")`.
  - The class must have a static `parse(String, ISelectorContext)` that
    returns an `ITargetSelectorDynamic` subtype. With `@SelectorAnnotation`
    it also needs `parse(IAnnotationHandle, ISelectorContext)`. Violations
    throw `MixinError`.
  - Registering a code again replaces the earlier entry.
  [TargetSelector.java:register; DynamicSelectorEntry; registerBuiltIn]
- A `MixinException` thrown inside a dynamic `parse` method is rethrown
  unwrapped. Other exceptions become
  `MixinError("Error parsing dynamic target selector [...] for ...")`
  [TargetSelector.java:DynamicSelectorEntry.parse].
- `parse(Iterable<?>)` accepts `IAnnotationHandle`, `AnnotationNode`, `String`
  and `Class` items. A `Class` becomes its type descriptor. Any other object
  is parsed from `toString()`, and `null` items are skipped. The result is a
  `LinkedHashSet`, so order is kept and equal selectors are merged
  [TargetSelector.java:parse(Iterable,ISelectorContext,Set)].
- `parseAndValidate(Iterable)` validates every parsed selector and stops at the
  first `InvalidSelectorException` [TargetSelector.java:parseAndValidate(Iterable,ISelectorContext)].
- `parseName` returns the selector's name when it is an
  `ITargetSelectorByName` with a non-null name. Otherwise it returns the input
  unchanged [TargetSelector.java:parseName].
- `run` counts only `isMatch()` results (`MATCH` or `EXACT_MATCH`). It stops
  once more than `getMaxMatchCount()` matches are seen, and keeps the first
  exact match. Fewer than `getMinMatchCount()` matches throw
  `SelectorConstraintException("%s did not match the required number of targets (required=%d, matched=%d)")`
  [TargetSelector.java:runSelector].
- `run` de-duplicates candidates with `candidates.contains(element)`. However,
  each `ElementNode` subclass delegates `equals` to the node it wraps, for
  example `this.method.equals(obj)`, so comparing one `ElementNode` with
  another is false. The de-duplication therefore never removes anything
  [TargetSelector.java:runSelector; ElementNode.java:ElementNodeMethod.equals].
- `Result.getSingleResult(strict)` returns the exact match when there is one.
  Otherwise it returns the first candidate, but only when there is exactly one
  candidate or `strict` is false. In the remaining cases it throws
  `IllegalStateException("No candidates were found")` or
  `IllegalStateException("Multiple candidates were found")`
  [TargetSelector.java:Result.getSingleResult].
- `MatchResult.isAtLeast(null)` is true. `isMatch()` is true for `MATCH` and
  `EXACT_MATCH` [MatchResult.java:isAtLeast; isMatch].
- `Configure.checkArgs` throws
  `IllegalArgumentException("Insufficient arguments for <NAME> mutation. Required <n> but received <m>")`
  [ITargetSelector.java:Configure.checkArgs].
- `MemberMatcher.parse` behaviour:
  - It reads `[owner|name|desc =] /regex/` groups. An unprefixed group is the
    name.
  - An invalid regex becomes `.*`, and its `PatternSyntaxException` is kept
    and printed with `printStackTrace()`.
  - A repeated group records
    `InvalidSelectorException("Pattern for '<x>' specified multiple times ...")`.
  - The recorded errors surface from `validate()`. `validate()` also throws
    when no group was found.
  [MemberMatcher.java:parse; validate]
- `MemberMatcher` limits and matching:
  - It has a minimum of 0 and a maximum of `Integer.MAX_VALUE`.
  - `next()` returns `this`, and `configure` returns `this` after
    `checkArgs`.
  - `match` gives `EXACT_MATCH` or `NONE` according to the last pattern whose
    input value is non-null. Earlier pattern results are overwritten.
  [MemberMatcher.java:getMaxMatchCount; next; matches]
- `TargetSelectors.parse` validates each selector and attaches it to the
  context, then converts errors to `InvalidInjectionException`:
  - `InvalidMemberDescriptorException` gives `"has invalid target descriptor"`;
  - `TargetNotSupportedException` gives `"specifies a target class '%s', which is not supported"`;
  - any other `InvalidSelectorException` gives `"is decorated with an invalid selector"`.
  [TargetSelectors.java:parse]
- `TargetSelectors.find()`:
  - Each selector is configured with `SELECT_MEMBER`, and only exact matches
    count.
  - A second, `PERMISSIVE` pass runs only when the permissive pass is enabled,
    the permissive selector differs, and the first pass matched nothing.
  [TargetSelectors.java:findRootTargets]
- When a selector's maximum count is 1 or less, every match is added.
  Otherwise a match is skipped when any of these holds:
  - it is static and the handler is not;
  - it is the handler method itself;
  - it carries `@MixinMerged`.

  A merged target that is `@Final` throws
  `InvalidInjectionException("%s cannot inject into @Final method ...")`
  [TargetSelectors.java:findRootTargets; checkTarget].
- When a selector matches fewer times than its minimum, `find()` throws
  `InvalidInjectionException`, whose cause is a `SelectorConstraintException`
  with the message `"Injection validation failed: ... did not match the required number of targets ..."`
  [TargetSelectors.java:findRootTargets].
- `validate(expected, required)` does nothing when there are targets. With no
  targets:
  - it throws `"Injection validation failed: ... could not find any targets matching ..."`
    when `DEBUG_INJECTORS` is on and `expected > 0`;
  - otherwise it throws `"Critical injection failure: ... could not find any targets matching ..."`
    when `required > 0`.
  [TargetSelectors.java:validate]
- `ElementNode` behaviour:
  - `of(ClassNode, TNode)` throws
    `IllegalArgumentException("Could not create ElementNode for unknown node type: ...")`.
  - `of(AbstractInsnNode)` returns `null` for anything other than method,
    field and invokedynamic instructions.
  - `dynamicInsnList` yields `null` for instructions other than
    `INVOKEDYNAMIC`.
  [ElementNode.java:of; ElementNodeIterator.next]
- For an `INVOKEDYNAMIC` with `bsmArgs.length > 1`, the node reads
  `bsmArgs[0..2]`. When the second argument is a `Handle`, the owner, the
  synthetic name and the descriptor come from that handle. The guard checks
  `length > 1` but the code reads index 2 [ElementNode.java:ElementNodeInvokeDynamicInsn].

## Side Effects

- Static registry: on class initialisation `TargetSelector` registers
  `DynamicSelectorDesc`. `register` prints a `WARNING` (override) or an
  `OTHER` (new) message through `MessageRouter.getMessager()`
  [TargetSelector.java:dynamicSelectors; register].
- `MemberMatcher.parse` prints a stack trace to `System.err` when a regex is
  invalid [MemberMatcher.java:parse].
- `TargetSelectors.find()` adds to its `targets` list, and `clear()` empties it.
  Neither changes the target class [TargetSelectors.java:findRootTargets; clear].

## Verification

- Compile: `mise exec -- ./gradlew compileJava` (run from the repository root).
- Test: `mise exec -- ./gradlew test`. `build.gradle` declares the `test`
  source set, but `src/test` does not exist yet, so there are no tests today.
