# struct

## Overview

The data structures behind injection:

- `InjectionInfo` and its subclasses are the per-handler metaclass for each
  injector annotation (`@Inject`, `@ModifyArg`, `@ModifyArgs`, `@Redirect`,
  `@ModifyVariable`, `@ModifyConstant`). `InjectionInfo` also keeps the static
  registry that maps annotations to metaclasses.
- `Target` and `Constructor` wrap a target `MethodNode`. They track injection
  nodes, local and stack growth, and the special points inside constructors.
- `InjectionNodes` holds injection nodes and their decorations.
- `MemberInfo` is the string member selector, for example
  `Lowner;name*(desc)ret`.
- `InjectionPointData` holds the parsed contents of an `@At`.
- `InjectorGroupInfo` implements `@Group` count validation.
- The annotation contexts give selectors their coordinates.

## Interface

- `public abstract class InjectionInfo extends SpecialMethodInfo implements ISliceContext`
  [InjectionInfo.java:InjectionInfo]
  - Nested meta-annotations, each with runtime retention and `TYPE` target:
    - `@AnnotationType(Class<? extends Annotation>)`;
    - `@HandlerPrefix(String)`, where `DEFAULT = "handler"`;
    - `@InjectorOrder(int)`, with `EARLY = 0`, `DEFAULT = 1000`, `LATE = 2000`,
      `REDIRECT = 10000` and `AFTER_REDIRECT = 20000`.
  - Lifecycle: `prepare()`, `preInject()`, `inject()` and `postInject()`.
  - Queries and callbacks: `isValid()`, `getOrder()`, `getTargetCount()`,
    `getInjectedCallbackCount()`, `getSlice(id)`, `getSliceId(id)`,
    `notifyInjected(Target)`, `addCallbackInvocation(MethodNode)`,
    `addMethod(access, name, desc)` and `addMessage(format, args)`.
  - Statics: `parse(MixinTargetContext, MethodNode)`,
    `getInjectorAnnotation(IMixinInfo, MethodNode)`,
    `getInjectorPrefix(AnnotationNode)`, `register(Class<? extends InjectionInfo>)`
    and `getRegisteredAnnotations()`.
- Built-in metaclasses:

  | Class | Annotation | Prefix | Order | Description |
  | --- | --- | --- | --- | --- |
  | `CallbackInjectionInfo` | `@Inject` | `handler` | `DEFAULT` | `"Callback method"` |
  | `ModifyArgInjectionInfo` | `@ModifyArg` | `modify` | `DEFAULT` | `"Argument modifier method"` |
  | `ModifyArgsInjectionInfo` | `@ModifyArgs` | `args` | `DEFAULT` | `"Multi-argument modifier method"` |
  | `RedirectInjectionInfo` | `@Redirect` | `redirect` | `REDIRECT` | `"Redirector"` |
  | `ModifyVariableInjectionInfo` | `@ModifyVariable` | `localvar` | `DEFAULT` | `"Variable modifier method"` |
  | `ModifyConstantInjectionInfo` | `@ModifyConstant` | `constant` | `REDIRECT` | `"Constant modifier method"` |

  [CallbackInjectionInfo.java; ModifyArgInjectionInfo.java; ModifyArgsInjectionInfo.java; RedirectInjectionInfo.java; ModifyVariableInjectionInfo.java; ModifyConstantInjectionInfo.java]
- `public class InjectorGroupInfo` [InjectorGroupInfo.java:InjectorGroupInfo]
  - Accessors: `getName`, `isDefault`, `getMinRequired`, `getMaxAllowed` and
    `getMembers`.
  - Mutators and checks: `setMinRequired`, `setMaxAllowed`, `add` and
    `validate()` (which throws `InjectionValidationException`).
  - Nested `public static final class Map extends HashMap<String, InjectorGroupInfo>`
    with `forName`, `parseGroup(MethodNode | AnnotationNode, String defaultGroup)`
    and `validateAll()`.
- `public class Target implements Comparable<Target>, Iterable<AbstractInsnNode>`
  [Target.java:Target]
  - Public final fields: `classInfo`, `classNode`, `method`, `insns`,
    `isStatic`, `arguments` and `returnType`.
  - `static Target of(ClassInfo, ClassNode, MethodNode)`. The constructor is
    package-private.
  - Injection nodes: `addInjectionNode` and `getInjectionNode`.
  - Locals and stack: `allocateLocal`, `allocateLocals`, `extendLocals`,
    `extendStack`, `getMaxLocals`, `getMaxStack`, `getCurrentMaxLocals` and
    `getCurrentMaxStack`.
  - Arguments and callbacks: `generateArgMap`, `getArgIndices`,
    `getCallbackInfoClass`, `getSimpleCallbackDescriptor` and
    `getCallbackDescriptor`.
  - Instruction edits: `insert`, `insertBefore`, `replaceNode`, `wrapNode`,
    `removeNode`, `addLocalVariable`, `findInitNodeFor`, `indexOf` and `get`.
  - The nested `public class Extension` provides `add`, `set`, `get` and
    `apply`.
- `public class Constructor extends Target` [Constructor.java:Constructor]
  - Instance methods: `findDelegateInitNode()`, `isInjectable()`,
    `inspect(Initialiser)`, `findInitialiserInjectionPoint(InjectionMode)` and
    `findFirstBodyInsn()`.
  - `static InsnRange getRange(MethodNode)`.
- `public class InjectionNodes extends ArrayList<InjectionNode>` provides
  `add(AbstractInsnNode)`, `get`, `contains`, `replace` and `remove`
  [InjectionNodes.java:InjectionNodes].
  - `InjectionNode` provides `getId`, `getOriginalTarget`, `getCurrentTarget`,
    `replace`, `remove`, `matches`, `isReplaced`, `isRemoved`, `decorate`,
    `hasDecoration` and `getDecoration(key[, default])`
    [InjectionNodes.java:InjectionNode].
- `public interface IChainedDecoration<T>` declares `void replace(T old)`
  [IChainedDecoration.java:IChainedDecoration].
- `public class ArgOffsets implements IChainedDecoration<ArgOffsets>`
  [ArgOffsets.java:ArgOffsets]
  - `KEY = "argOffsets"` and the static `DEFAULT`.
  - `ArgOffsets(int offset, int length)`.
  - `getLength`, `isEmpty`, `getStartIndex`, `getEndIndex`,
    `getArgIndex(int[, boolean])` and `apply(Type[])`.
- `public final class MemberInfo implements ITargetSelectorRemappable, ITargetSelectorConstructor`
  [MemberInfo.java:MemberInfo]
  - Factories: `static parse(String, ISelectorContext)` and
    `static fromMapping(IMapping)`.
  - Selector methods: `validate`, `match`, `matches`, `configure`, `attach`,
    `move`, `transform` and `remapUsing`.
  - Mapping and conversion: `asMapping`, `asMethodMapping`, `asFieldMapping`,
    `toDescriptor`, `toCtorType`, `toCtorDesc` and the deprecated `toSrg`.
  - Queries: `isField`, `isConstructor`, `isClassInitialiser`,
    `isInitialiser` and `isFullyQualified`.
- `public class InvalidMemberDescriptorException extends InvalidSelectorException`
  provides `getInput()`. `public class TargetNotSupportedException extends InvalidSelectorException`
  [InvalidMemberDescriptorException.java; TargetNotSupportedException.java].
- `public class InjectionPointData` [InjectionPointData.java:InjectionPointData]
  - Parsed `@At` values: `getAt`, `getType`, `getSpecifier`, `getSlice`,
    `getOrdinal`, `getOpcode()`, `getOpcode(default)`,
    `getOpcode(default, valid...)`, `getOpcodeList`, `getId` and `isUnsafe`.
  - Context: `getTargetRestriction`, `getContext`, `getMixin`, `getMethod`,
    `getMethodReturnType`, `getParent` and `getDescription`.
  - Typed arguments: `get(key, String | int | boolean | Enum)` and
    `get(key)`, which returns an `ITargetSelector`.
  - `getTarget()`, `getLocalVariableDiscriminator()` and
    `static parseType(String)`.
- `public class SelectorAnnotationContext implements ISelectorContext`, and
  `public class InjectionPointAnnotationContext extends SelectorAnnotationContext implements IInjectionPointContext`
  [SelectorAnnotationContext.java; InjectionPointAnnotationContext.java].

## Invariants

- The registry is a static `LinkedHashMap` keyed by annotation descriptor.
  - `register` throws `IllegalArgumentException("Injection info class ... is not annotated with @AnnotationType")`
    when `@AnnotationType` is missing.
  - It throws `MixinError("InjectionInfo class ... is missing a valid constructor")`
    when the class has no `(MixinTargetContext, MethodNode, AnnotationNode)`
    constructor.
  - Registering the same annotation again replaces the earlier entry.
  [InjectionInfo.java:register]
- `parse` returns `null` when no registered annotation is present.
  `getInjectorAnnotation` turns an `IllegalArgumentException` from
  `Annotations.getSingleVisible`, raised when there are several injector
  annotations, into `InvalidMixinException("Error parsing annotations on ...")`
  [InjectionInfo.java:parse; getInjectorAnnotation].
- `InjectorEntry.create` rethrows a `MixinException` cause directly. Any other
  constructor failure is wrapped in `MixinError("Error initialising injector metaclass ...")`
  or `MixinError("Failed to instantiate injector metaclass ...")`
  [InjectionInfo.java:InjectorEntry.create].
- `getInjectorPrefix` matches with `desc.endsWith(annotationDesc)`. It returns
  `"handler"` for a `null` annotation or no match [InjectionInfo.java:getInjectorPrefix].
- The constructor reads the annotation in this order:
  1. injection points (`at`);
  2. requirements;
  3. order;
  4. selectors (`method` and `target`);
  5. the target search;
  6. target validation against `expect` and `require`;
  7. injection point parsing;
  8. the injector.

  An `InvalidMixinException` gets the activity stack prepended. Any other
  exception is wrapped as `InvalidMixinException("Unexpected <Type> parsing ...")`
  [InjectionInfo.java:readAnnotation].
- A missing `at` key does not throw, because `Annotations.getValue(annotation, key, false)`
  returns an empty list (see src/main/java/org/spongepowered/asm/util/Annotations.java).
  `@ModifyConstant` reads `constant` instead of `at`. When that list is empty it
  adds one synthetic `@Constant(log = true)`
  [InjectionInfo.java:readInjectionPoints; ModifyConstantInjectionInfo.java:readInjectionPoints].
- When `method` and `target` both yield no selectors, the constructor throws
  `InvalidInjectionException("%s is missing 'method' or 'target' to specify targets")`
  [InjectionInfo.java:parseSelectors].
- Requirements:
  - `expect` defaults to 1.
  - `require` is used when it is above -1. Otherwise, when the injector has no
    `@Group`, it becomes `mixin.getDefaultRequiredInjections()`. In all other
    cases it stays 0.
  - `allow`, when set, becomes `max(max(require, 1), allow)`. Without it the
    limit is `Integer.MAX_VALUE`.
  [InjectionInfo.java:parseRequirements]
- `order` from the annotation overrides the class's `@InjectorOrder`. The
  fallback is `InjectorOrder.DEFAULT` [InjectionInfo.java:parseOrder].
- `isValid()` requires at least one target and at least one injection point
  [InjectionInfo.java:isValid].
- `prepare()` converts a `SelectorException` thrown by `injector.find` into
  `InvalidInjectionException("Injection validation failed: ...")`. It always
  disposes the `InjectorTarget` [InjectionInfo.java:prepare].
- `postInject()` applies three checks in this order:
  1. With `DEBUG_INJECTORS` on and fewer injections than `expect`, it throws
     `InvalidInjectionException("Injection validation failed: ... expected %d invocation(s) but %d succeeded ...")`.
  2. With fewer injections than `require`, it throws
     `InjectionError("Critical injection failure: ... failed injection check, (%d/%d) succeeded ...")`.
  3. With more injections than `allow`, it throws
     `InjectionError("Critical injection failure: ... %d succeeded of %d allowed.")`.
  [InjectionInfo.java:postInject]
- `getSliceId` returns `""` in the base class, so every slice id maps to the
  default slice. Only `CallbackInjectionInfo` and `ModifyConstantInjectionInfo`
  keep slice ids, through `Strings.nullToEmpty(id)`
  [InjectionInfo.java:getSliceId; CallbackInjectionInfo.java:getSliceId].
- `addMethod` always adds `ACC_SYNTHETIC` [InjectionInfo.java:addMethod].
- `InjectorGroupInfo`:
  - `setMinRequired` and `setMaxAllowed` throw `IllegalArgumentException` for
    values below 1.
  - A conflicting value logs a WARN. The strictest value is kept: the larger
    minimum and the smaller maximum.
  - `getMinRequired()` is at least 1.
  - `validate()` skips empty groups and checks the sum of the members'
    injected counts. Too few throw
    `InjectionValidationException("expected %d invocation(s) but [only ]%d succeeded")`.
    Too many throw `"maximum of %d invocation(s) allowed but %d succeeded"`.
  [InjectorGroupInfo.java:setMinRequired; setMaxAllowed; validate]
- `Map.parseGroup` behaviour:
  - Without `@Group` it returns the shared static `NO_GROUP`. That group's
    `isDefault()` is true and it is not stored in the map.
  - An empty `name` uses `defaultGroup`.
  - A `min` or `max` of -1 is ignored.
  - `Map.get(key)` creates the group when it is missing.
  [InjectorGroupInfo.java:Map.parseGroup; Map.get]
- `Target.Extension.apply()` sets the method's max to
  `max(current, original + size)`, where `original` is the value captured when
  the `Target` was created. It then resets `size` to 0. In contrast,
  `allocateLocals(n)` returns the current `maxLocals` and increases it
  directly [Target.java:Extension.apply; extendLocalsBy; allocateLocals].
- `generateArgMap(args, start, fresh)` leaves entries before `start` at 0. With
  `fresh = false` it reuses earlier arg-map locals [Target.java:generateArgMap; allocateArgMapLocal].
- `getArgIndices()` begins at slot 0 for static methods and slot 1 otherwise,
  and steps by each argument's size [Target.java:calcArgIndices].
- The basic callback descriptor is `(<target args>L<CallbackInfo class>;)V`.
  With captured locals, the non-null locals from `startIndex` onwards are
  appended, up to `extra` of them [Target.java:getCallbackDescriptor].
- `compareTo` orders by `toString()`, which is `owner::name desc`. A `null`
  argument gives `Integer.MAX_VALUE` [Target.java:compareTo].
- `Target.of` returns a `Constructor` when the method name is `<init>`
  [Target.java:of].
- `addLocalVariable` replaces an entry with the same index and the same
  start and end labels. The new entry is a `SyntheticLocalVariableNode`
  [Target.java:addLocalVariable].
- `Constructor.isInjectable()` is true when there is no delegate call or the
  delegate is `super(...)`. `inspect` throws
  `IllegalStateException("Attempted to inspect an incoming initialiser after the injection point was already determined")`
  once the injection point is known [Constructor.java:isInjectable; inspect].
- `findInitialiserInjectionPoint` searches for the last `<init>` call on this
  class or its superclass. In `SAFE` mode it uses the first such call instead.
  In `DEFAULT` mode it also considers the last `PUTFIELD` of a field that a
  mixin initialises. It returns `null` when nothing qualifies. The first result
  is cached whatever mode is passed later [Constructor.java:findInitialiserInjectionPoint].
- `InjectionNodes.add` returns the existing node when one matches the original
  or current instruction. `replace` and `remove` change only the
  `currentTarget`. `isReplaced()` is `original != current`, and `isRemoved()`
  is `current == null` [InjectionNodes.java:add; InjectionNode].
- Node and `InsnListEx` decorations behave the same way: when an
  `IChainedDecoration` replaces a value of the same class, it receives the old
  value through `replace(old)`. `hasDecoration` is false for `null` values
  [InjectionNodes.java:InjectionNode.decorate].
- `ArgOffsets` rules:
  - `getArgIndex(i)` returns `i + offset`, then passes the result to any
    chained earlier `ArgOffsets`.
  - With `mustBeInWindow`, it throws `IndexOutOfBoundsException` only when
    `index > length`. An index equal to `length` is allowed.
  - `apply` returns an array of `length` entries. Entries beyond the input are
    `null`.
  - `DEFAULT` maps each index to itself and returns its input unchanged.
  [ArgOffsets.java:getArgIndex; apply; Default]
- `MemberInfo.parse(input, context)` reads the input in this order:
  1. It removes all whitespace.
  2. It splits on `->` into the selector and a tail.
  3. It remaps the selector through `context.remap`.
  4. It reads the owner: the text before the last `.`, or `Lowner;`.
  5. It reads the descriptor: from `(`, or after `:` for a field.
  6. It reads a trailing quantifier: `*` (any), `+` (one or more) or `{...}`.
     A malformed `{` gives `Quantifier.NONE`.
  7. An empty name becomes `null`.
  [MemberInfo.java:parse]
- Constructing a `MemberInfo` with an owner that contains `.` throws
  `IllegalArgumentException("Attempt to instance a MemberInfo with an invalid owner format")`.
  `MemberInfo(AbstractInsnNode)` accepts only method and field instructions
  [MemberInfo.java:MemberInfo].
- `validate()` throws `InvalidMemberDescriptorException` in these cases:
  - `"Malformed quantifier in selector: ..."` when the maximum count is 0;
  - `"Invalid owner: ..."` for an owner with illegal characters;
  - `"Malformed owner: ..."` for a dotted input whose owner starts with `L`;
  - `"Invalid name: ..."`;
  - `"Invalid descriptor: ..."`;
  - `"Invalid field type in descriptor: ..."`;
  - `"Invalid return type \"...\" in descriptor: ..."`.
  [MemberInfo.java:validate]
- `matches(owner, name, desc)`:
  - A descriptor or owner that differs, with both sides non-null, gives
    `NONE`.
  - An equal name gives `EXACT_MATCH`, a name equal ignoring case gives
    `MATCH`, and any other name gives `NONE`.
  - A `null` name on either side gives `EXACT_MATCH`.
  [MemberInfo.java:matches]
- `attach(context)` throws `TargetNotSupportedException(owner)` when the owner
  is set and differs from the mixin's target class [MemberInfo.java:attach].
- `configure` returns a new selector in these cases:
  - `SELECT_MEMBER` gives `SINGLE` and `SELECT_INSTRUCTION` gives `ANY`, but
    only when the quantifier is the default.
  - `MOVE` and `ORPHAN` change the owner.
  - `TRANSFORM` and `PERMISSIVE` change the descriptor. `PERMISSIVE` sets it to
    `null`.
  - `CLEAR_LIMITS` gives `ANY`.
  [MemberInfo.java:configure]
- Other `MemberInfo` conversions:
  - `next()` parses the `->` tail with a `null` context.
  - `toCtorType()` returns `null` unless the selector came from `parse`.
  - `asMethodMapping`, `asFieldMapping` and `toSrg` throw `MixinException` for
    unqualified selectors or the wrong member kind.
  - `equals` compares the quantifier, the field flag, the owner, the name and
    the descriptor.
  [MemberInfo.java:next; toCtorType; asMethodMapping; equals]
- `InjectionPointData` parsing:
  - The `@At` value is matched against `^(.+?)(:(ALL|FIRST|LAST|ONE|DEFAULT))?$`.
    A specifier is recognised only in upper case. Without one the specifier is
    `DEFAULT`.
  - `ordinal` is clamped to at least -1.
  - `unsafe` gives `RestrictTargetLevel.ALLOW_ALL`. Otherwise the level is
    `METHODS_ONLY`.
  [InjectionPointData.java:createPattern; InjectionPointData]
- Each `args` entry is split at its first `=`. A key with no `=` maps to `""`.
  After parsing, the constructor writes the `@At` values `target`, `ordinal`
  and `opcode` into the same map, so they override any `args` entries with
  those names [InjectionPointData.java:parseArgs; InjectionPointData].
- Typed `get` helpers fall back to the default when parsing fails. Boolean
  parsing never fails, so any value other than `true` reads as `false`. Enum
  values must match the constant names exactly
  [InjectionPointData.java:parseInt; parseBoolean; parseEnum].
- `getOpcode(default, valid...)` returns the opcode only when it is in
  `valid`. `getOpcode(default)` returns it when it is above 0.
  `getOpcodeList` splits on space, comma or semicolon and returns a sorted
  set of opcode names that parse to a positive value
  [InjectionPointData.java:getOpcode; getOpcodeList].
- `getTarget()` and `get(key)` wrap an `InvalidSelectorException` as
  `InvalidInjectionPointException("Failed validating @At(\"%s\").target ...")`
  or `"Failed parsing @At(\"%s\").%s ..."`. With an empty `target`, `getTarget()`
  uses the selector annotation's `desc` when its `id` is `"at"`, which is the
  default [InjectionPointData.java:getTarget; get(String)].
- `SelectorAnnotationContext` passes `getMixin`, `getMethod`, `getAnnotation`
  and `remap` to its parent. `getSelectorCoordinate(leaf)` returns the fixed
  coordinate whatever `leaf` is. `InjectionPointAnnotationContext` also passes
  `addMessage`, `getMethod` and `getAnnotationNode` to its parent context
  [SelectorAnnotationContext.java; InjectionPointAnnotationContext.java].

## Side Effects

- Static state:
  - The `InjectionInfo` registry and its `registeredAnnotations` array are
    filled in a static initialiser with the six built-ins.
  - `register` prints a `WARNING` (override) or an `OTHER` (new) message
    through `MessageRouter.getMessager()`.
  [InjectionInfo.java:registry; register]
- `InjectionNode.nextId` is a static counter. It is incremented for every node
  created in the JVM and is not synchronised [InjectionNodes.java:InjectionNode.nextId].
- `InjectorGroupInfo.Map.NO_GROUP` is a static group shared by all maps.
  Every ungrouped injector is added to its member list, which is never
  cleared [InjectorGroupInfo.java:Map.NO_GROUP; InjectionInfo.java:parseRequirements].
- `ArgOffsets.DEFAULT` is a mutable `public static` field [ArgOffsets.java:DEFAULT].
- `postInject()` adds methods created by `addMethod` to the target
  `classNode.methods` [InjectionInfo.java:postInject].
- `inject()` clears the selected targets once it finishes [InjectionInfo.java:inject].
- `Target` changes the `MethodNode` it wraps:
  - `maxLocals` and `maxStack`;
  - the instruction list, including lazily inserted start and end `LabelNode`s
    for local variables;
  - `localVariables`.
  [Target.java:allocateLocals; addLocalVariable; getStartLabel]
- `Constructor` inserts `MarkerNode`s (`INITIALISER_TAIL`, `BODY_START`) into
  the instruction list. `getRange` may insert a `LabelNode` and a
  `LineNumberNode` before the final `RETURN` [Constructor.java:findInitialiserInjectionPoint; findFirstBodyInsn; getRange].
- `InjectorGroupInfo` logs conflicting `min` and `max` values at WARN on the
  `"mixin"` logger [InjectorGroupInfo.java:setMinRequired].

## Verification

- Compile: `mise exec -- ./gradlew compileJava` (run from the repository root).
- Test: `mise exec -- ./gradlew test`. `build.gradle` declares the `test`
  source set, but `src/test` does not exist yet, so there are no tests today.
