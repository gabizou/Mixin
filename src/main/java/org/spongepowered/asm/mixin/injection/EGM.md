# injection

## Overview

The public injector API of Mixin, and the injection point framework under it.
This directory declares:

- the injector annotations: `@Inject`, `@Redirect`, `@ModifyArg`,
  `@ModifyArgs`, `@ModifyVariable` and `@ModifyConstant`;
- the annotations that support them: `@At`, `@Slice`, `@Constant`, `@Desc`,
  `@Descriptors`, `@Next`, `@Group`, `@Coerce` and `@Surrogate`;
- `InjectionPoint`, the abstract base of every `@At` point, with its static
  registry, its parser and its combinators.

The subpackages hold the metaclasses (`struct`), the injector implementations
(`callback`, `invoke`, `modify`), the built-in injection points (`points`), the
selector system (`selectors`), the instruction-list views (`code`) and the
exceptions (`throwables`). Registration of custom injection points and
selectors from mixin configs is in
src/main/java/org/spongepowered/asm/mixin/transformer/MixinConfig.java.
Parsing of the `constraints` element is in
src/main/java/org/spongepowered/asm/util/ConstraintParser.java.

## Interface

### Injector annotations

All of these have runtime retention and target methods.

| Annotation | Elements and defaults |
| --- | --- |
| `@Inject` | `id = ""`, `method = {}`, `target = {}` (`@Desc[]`), `slice = {}` (`@Slice[]`), `at` (`@At[]`, required), `cancellable = false`, `locals = NO_CAPTURE`, `remap = true`, `require = -1`, `expect = 1`, `allow = -1`, `constraints = ""`, `order = 1000` [Inject.java:Inject] |
| `@Redirect` | `method`, `target`, `slice = @Slice` (one), `at` (one, required), `remap`, `require = -1`, `expect = 1`, `allow = -1`, `constraints`, `order = 10000` [Redirect.java:Redirect] |
| `@ModifyArg` | the `@Redirect` elements plus `index = -1`, with `order = 1000` [ModifyArg.java:ModifyArg] |
| `@ModifyArgs` | the `@Redirect` elements, with `order = 1000` [ModifyArgs.java:ModifyArgs] |
| `@ModifyVariable` | the `@Redirect` elements plus `print = false`, `ordinal = -1`, `index = -1`, `name = {}`, `argsOnly = false`, with `order = 1000` [ModifyVariable.java:ModifyVariable] |
| `@ModifyConstant` | `method`, `target`, `slice = {}` (`@Slice[]`), `constant = {}` (`@Constant[]`), `remap`, `require = -1`, `expect = 1`, `allow = -1`, `constraints`, `order = 10000` [ModifyConstant.java:ModifyConstant] |

### Supporting annotations

- `@At` has runtime retention and no element target, so it is used only
  inside other annotations [At.java:At]. Its elements:
  - `value` (required);
  - `id = ""`, `slice = ""`, `shift = Shift.NONE`, `by = 0`, `args = {}`,
    `target = ""`, `desc = @Desc("")`, `ordinal = -1`, `opcode = -1`,
    `remap = true` and `unsafe = false`.
  - `enum Shift` has the constants `NONE`, `BEFORE`, `AFTER` and `BY`.
- `@Slice` has the elements `id = ""`, `from = @At("HEAD")` and
  `to = @At("TAIL")` [Slice.java:Slice].
- `@Constant` [Constant.java:Constant]
  - Elements: `nullValue = false`, `intValue = 0`, `floatValue = 0.0F`,
    `longValue = 0L`, `doubleValue = 0.0`, `stringValue = ""`,
    `classValue = Object.class`, `ordinal = -1`, `slice = ""`,
    `expandZeroConditions = {}` and `log = false`.
  - `enum Condition` has the constants `LESS_THAN_ZERO`,
    `LESS_THAN_OR_EQUAL_TO_ZERO`, `GREATER_THAN_OR_EQUAL_TO_ZERO` and
    `GREATER_THAN_ZERO`, with the methods `getEquivalentCondition()` and
    `getOpcodes()`.
- `@Desc` has runtime retention, targets types and methods, and is
  `@Repeatable(Descriptors.class)`. Its elements are `id = ""`,
  `owner = void.class`, `value` (required), `ret = void.class`, `args = {}`,
  `next = {}` (`@Next[]`), `min = 0` and `max = Integer.MAX_VALUE`
  [Desc.java:Desc].
- `@Descriptors(Desc[] value)` targets types and methods
  [Descriptors.java:Descriptors].
- `@Next` has the elements `name = ""`, `ret = void.class`, `args = {}`,
  `min = 0` and `max = Integer.MAX_VALUE` [Next.java:Next].
- `@Group` has `RetentionPolicy.CLASS` and targets methods. Its elements are
  `name = ""`, `min = -1` and `max = -1` [Group.java:Group].
- `@Coerce` targets parameters and methods and declares no `@Retention`, so it
  uses the default `CLASS` retention [Coerce.java:Coerce].
- `@Surrogate` has runtime retention and targets methods
  [Surrogate.java:Surrogate].

### Injection points

- `public interface IInjectionPointContext extends IMessageSink, ISelectorContext`
  declares `MethodNode getMethod()` and `AnnotationNode getAnnotationNode()`
  [IInjectionPointContext.java:IInjectionPointContext].
- `public abstract class InjectionPoint` [InjectionPoint.java:InjectionPoint]
  - Nested types:
    - `@AtCode(value, namespace = "")`;
    - `enum Specifier { ALL, FIRST, LAST, ONE, DEFAULT }`;
    - `enum RestrictTargetLevel { METHODS_ONLY, CONSTRUCTORS_AFTER_DELEGATE, ALLOW_ALL }`;
    - `Flags.UNSAFE = 1`, with `Flags.parse(At | AnnotationNode)`.
  - Constants: `DEFAULT_ALLOWED_SHIFT_BY = 0` and `MAX_ALLOWED_SHIFT_BY = 5`.
  - Instance methods: `getSlice()`, `getSpecifier(Specifier default)`,
    `getId()`, `checkPriority(int targetPriority, int mixinPriority)`,
    `getTargetRestriction(IInjectionPointContext)`,
    `abstract boolean find(String desc, InsnList insns, Collection<AbstractInsnNode> nodes)`
    and the protected `addMessage`, `setTargetRestriction`, `getAtCode` and
    `nextNode`.
  - Static combinators: `and(...)`, `or(...)`, `after(point)`,
    `before(point)` and `shift(point, count)`.
  - Static parsers: `parse(...)` overloads for `At`, `AnnotationNode`,
    `List<AnnotationNode>` and raw strings.
  - Static registration: `register(Class, String namespace)` and the
    deprecated `register(Class)`.

### Subdirectories

- [callback](./callback/EGM.md): `@Inject` runtime types (`CallbackInfo`,
  `CallbackInfoReturnable`, `Cancellable`, `CancellationException`),
  `LocalCapture`, and `CallbackInjector`.
- [code](./code/EGM.md): the `Injector` base class (find and inject cycle,
  signature and `@Coerce` validation), `InjectorTarget`, `MethodSlice` and
  `MethodSlices`, and the read-only `InsnListEx` views.
- [invoke](./invoke/EGM.md): `RedirectInjector`, `ModifyArgInjector`,
  `ModifyArgsInjector` and `ModifyConstantInjector`, with the `arg` (`Args`
  generation) and `util` (`InsnFinder`) subpackages.
- [modify](./modify/EGM.md): `ModifyVariableInjector`,
  `LocalVariableDiscriminator`, and the `LOAD` and `STORE` injection points.
- [points](./points/EGM.md): the built-in `@At` points `HEAD`, `CTOR_HEAD`,
  `RETURN`, `TAIL`, `INVOKE`, `INVOKE_ASSIGN`, `INVOKE_STRING`, `FIELD`, `NEW`,
  `JUMP` and `CONSTANT`.
- [selectors](./selectors/EGM.md): `ITargetSelector`, `TargetSelector` (parsing
  and the dynamic registry), `TargetSelectors`, `MemberMatcher`, `ElementNode`
  and `MatchResult`, with the `dynamic` (`@Desc`) and `throwables`
  subpackages.
- [struct](./struct/EGM.md): `InjectionInfo` (the metaclass registry and its
  lifecycle), `Target` and `Constructor`, `InjectionNodes`, `MemberInfo`,
  `InjectionPointData`, `InjectorGroupInfo` and `ArgOffsets`.
- [throwables](./throwables/EGM.md): `InjectionError`,
  `InjectionValidationException`, `InvalidInjectionException`,
  `InvalidInjectionPointException` and `InvalidSliceException`.

## Invariants

- On class initialisation, the static type registry maps 13 built-in points.
  Each is stored under its upper-case code and under `MIXIN:<code>`:
  `FIELD`, `INVOKE`, `NEW`, `RETURN`, `INVOKE_STRING`, `JUMP`, `HEAD`,
  `INVOKE_ASSIGN`, `LOAD`, `STORE`, `TAIL`, `CONSTANT` and `CTOR_HEAD`
  [InjectionPoint.java:static initialiser; registerBuiltIn].
- `register(type, namespace)` rules:
  - It throws `IllegalArgumentException("Injection point class ... is not annotated with @AtCode")`
    when `@AtCode` is missing.
  - A non-empty `@AtCode.namespace` overrides the `namespace` argument.
  - The type is stored under `NAMESPACE:CODE`, upper-cased, or under `CODE`
    when there is no namespace.
  - It logs at DEBUG when the code replaces a different class. Registering
    without a namespace logs a deprecation WARN.
  [InjectionPoint.java:register(Class,String)]
- Point lookup and creation:
  - The parsed type is upper-cased before the registry lookup.
  - An unknown type that looks like a fully qualified class name is loaded
    through `MixinService.getService().getClassProvider().findClass` and
    cached. If loading fails, it throws
    `InvalidInjectionException("... could not be loaded or is not a valid InjectionPoint")`.
  - Any other unknown type throws
    `InvalidInjectionException("... is not a valid injection point specifier")`.
  [InjectionPoint.java:findClass]
- Each point class needs a constructor that takes `InjectionPointData`. It is
  made accessible and invoked reflectively.
  - A class without one throws
    `InvalidInjectionException("... must contain a constructor which accepts an InjectionPointData")`.
  - An exception thrown by that constructor is wrapped as
    `InvalidInjectionException("Error whilst instancing injection point ... for ...", cause)`.
  [InjectionPoint.java:create]
- `parse(IInjectionPointContext, AnnotationNode)` uses these defaults for keys
  absent from bytecode: `target = ""`, `slice = ""`, `shift = NONE`, `by = 0`,
  `ordinal = -1`, `opcode = 0`, `id = null` and an empty `args` list. Note that
  `@At.opcode` declares -1 as its default
  [InjectionPoint.java:parse(IInjectionPointContext,AnnotationNode)].
- `parse(context, List<AnnotationNode>)` creates an
  `InjectionPointAnnotationContext` with the coordinate `"at"` for each
  `@At`. It skips `null` results and returns an immutable list
  [InjectionPoint.java:parse(IInjectionPointContext,List)].
- Shifts:
  - `Shift.BEFORE` gives `Shift(point, -1)`, `AFTER` gives `+1`, and `BY`
    gives `+by`.
  - A shifted index outside the list is dropped, and the message
    `"@At.shift offset outside the target bounds ..."` is added to the input
    point's message sink.
  - When the caller's `nodes` collection is not a `List`, the shift copies it
    before calling the input point, so nodes the input adds are not kept.
  [InjectionPoint.java:shift; Shift.find]
- `Shift.BY` validation reads `Option.SHIFT_BY_VIOLATION_BEHAVIOUR`, whose
  default is `WARN`:
  - `IGNORE` skips the check.
  - The allowed value is `MixinTargetContext.getMaxShiftByValue()`, or 0 for
    other contexts. The config clamps that value to the range 0 to 5 (see
    src/main/java/org/spongepowered/asm/mixin/transformer/MixinConfig.java).
  - `by` above the allowed value but at most 5 logs a WARN when the behaviour
    is `WARN`, and otherwise throws `InvalidInjectionException`.
  - Unless the behaviour is `IGNORE`, `by > 5` always throws
    `InvalidInjectionException(... "exceeds MAX_ALLOWED_SHIFT_BY=5" ...)`.
  [InjectionPoint.java:validateByValue]
- The default `checkPriority` returns `targetPriority < mixinPriority`
  [InjectionPoint.java:checkPriority].
- The no-argument constructor uses slice `""`, `Specifier.DEFAULT`, a `null`
  id, no message sink and `METHODS_ONLY`. The `InjectionPointData`
  constructor copies all of these from the data
  [InjectionPoint.java:InjectionPoint()].
- `getSpecifier(default)` returns `default` when the specifier is `DEFAULT`
  [InjectionPoint.java:getSpecifier].
- `getAtCode()` returns the `@AtCode` value in upper case, or the class name
  when there is no `@AtCode`. `toString()` is `@At("<code>")`
  [InjectionPoint.java:getAtCode; toString].
- Composite points:
  - The constructor needs at least two components, or it throws
    `IllegalArgumentException("Must supply two or more component injection points for composite point!")`.
  - The target restriction is the most permissive component level.
  - `Union` keeps first-seen order with no duplicates.
  - In `Intersection`, the flag `in` is never set to `false`, so every node
    from the first component is returned.
  [InjectionPoint.java:CompositeInjectionPoint; Union.find; Intersection.find]
- A `Shift` with a `null` input throws
  `IllegalArgumentException("Must supply an input injection point for SHIFT")`
  [InjectionPoint.java:Shift].
- `Constant.Condition`:
  - `LESS_THAN_ZERO` maps to `IFLT` and `IFGE`.
  - `LESS_THAN_OR_EQUAL_TO_ZERO` maps to `IFLE` and `IFGT`.
  - `GREATER_THAN_OR_EQUAL_TO_ZERO` and `GREATER_THAN_ZERO` are equivalent to
    the first two, and share their opcodes.
  [Constant.java:Condition]
- `@Group` and `@Coerce` have `CLASS` retention and are read as invisible
  annotations. The injector annotations, `@Desc` and `@Surrogate` have
  `RUNTIME` retention and are read as visible annotations
  [Group.java:Group; Coerce.java:Coerce].

## Side Effects

- Static state: the injection point registry is a static `HashMap`. It is
  filled when the class initialises, and grows through `register` and
  through `findClass` caching [InjectionPoint.java:types].
- Class loading: `findClass` loads custom injection point classes through the
  active `MixinService` class provider [InjectionPoint.java:findClass].
- Logging: `register` and `validateByValue` log through
  `MixinService.getService().getLogger("mixin")` [InjectionPoint.java:register; validateByValue].
- `Shift.find` changes the caller's `nodes` collection in place
  [InjectionPoint.java:Shift.find].

## Verification

- Compile: `mise exec -- ./gradlew compileJava` (run from the repository root).
- Test: `mise exec -- ./gradlew test`. `build.gradle` declares the `test`
  source set, but `src/test` does not exist yet, so there are no tests today.
