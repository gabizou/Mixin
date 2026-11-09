# points

## Overview

The built-in `@At` injection points. Each class extends `InjectionPoint`
(src/main/java/org/spongepowered/asm/mixin/injection/InjectionPoint.java) and
carries an `@AtCode`. Each implements
`find(String desc, InsnList insns, Collection<AbstractInsnNode> nodes)`, which
adds the matching instructions to `nodes` and returns whether any were found.
`InjectionPoint` registers all of them as built-ins, under the bare code and
under `MIXIN:<code>`. Named arguments come from `@At(args = "key=value")`
through `InjectionPointData`
(src/main/java/org/spongepowered/asm/mixin/injection/struct/InjectionPointData.java).

## Interface

| Code | Class | Selects |
| --- | --- | --- |
| `HEAD` | `MethodHead` | the first instruction of the list [MethodHead.java:find] |
| `CTOR_HEAD` | `ConstructorHead extends MethodHead` | a point in the constructor after the delegate call or initialisers, see Invariants [ConstructorHead.java:find] |
| `RETURN` | `BeforeReturn` | each return instruction whose opcode matches the method's return type [BeforeReturn.java:find] |
| `TAIL` | `BeforeFinalReturn` | the last return instruction that matches the return type [BeforeFinalReturn.java:find] |
| `INVOKE` | `BeforeInvoke` | `MethodInsnNode`s matched by the `target` selector [BeforeInvoke.java:find] |
| `INVOKE_ASSIGN` | `AfterInvoke extends BeforeInvoke` | the instruction after a non-void call, or after the store that follows it [AfterInvoke.java:addInsn] |
| `INVOKE_STRING` | `BeforeStringInvoke extends BeforeInvoke` | a call taking `(Ljava/lang/String;)V` that comes directly after a matching `LDC` [BeforeStringInvoke.java:inspectInsn] |
| `FIELD` | `BeforeFieldAccess extends BeforeInvoke` | `FieldInsnNode`s matched by the target selector, optionally only those with a following array access [BeforeFieldAccess.java:matchesInsn] |
| `NEW` | `BeforeNew` | `NEW` type instructions, optionally only those with a matching `<init>` [BeforeNew.java:find] |
| `JUMP` | `JumpInsnPoint` | `JumpInsnNode`s, optionally filtered by opcode [JumpInsnPoint.java:find] |
| `CONSTANT` | `BeforeConstant` | constant-loading instructions, or expanded zero-comparison jumps [BeforeConstant.java:find] |

- `BeforeInvoke` [BeforeInvoke.java:BeforeInvoke]
  - `public enum SearchType { STRICT, PERMISSIVE }`.
  - `BeforeInvoke setLogging(boolean)`.
  - Protected hooks `addInsn`, `matchesInsn`, `inspectInsn`, `matchesOrdinal`
    and `log`.
- `AfterInvoke.DEFAULT_SKIP` is a `public static final int[]` of
  arithmetic, conversion, `DUP`, `CHECKCAST` and `INSTANCEOF` opcodes
  [AfterInvoke.java:DEFAULT_SKIP].
- `BeforeFieldAccess` [BeforeFieldAccess.java:BeforeFieldAccess]
  - `ARRAY_SEARCH_FUZZ_DEFAULT = 8`.
  - `getFuzzFactor()` and `getArrayOpcode()`.
  - `static AbstractInsnNode findArrayNode(InsnList, FieldInsnNode, int opcode, int searchRange)`.
- `BeforeNew` [BeforeNew.java:BeforeNew]
  - `hasDescriptor()` and `getDescriptor()`.
  - `static MethodInsnNode findInitNodeFor(InsnList, TypeInsnNode, String desc)`.
- `BeforeConstant` [BeforeConstant.java:BeforeConstant(IMixinContext,AnnotationNode,String)]
  - `BeforeConstant(IMixinContext, AnnotationNode, String returnType)` reads a
    `@Constant` annotation.
  - `BeforeConstant(InjectionPointData)` reads `@At` arguments.

## Invariants

- `ordinal` defaults to -1, which means every match. Otherwise only the match
  at that zero-based ordinal is taken. `RETURN`, `JUMP`, `NEW`, `CONSTANT` and
  the `INVOKE` family use it [BeforeReturn.java:find; JumpInsnPoint.java:find].
- `HEAD`, `RETURN` and `TAIL` override `checkPriority` to always return
  `true`. Other points use the base rule `targetPriority < mixinPriority`
  [MethodHead.java:checkPriority; BeforeReturn.java:checkPriority; BeforeFinalReturn.java:checkPriority].
- `RETURN` and `TAIL` return `RestrictTargetLevel.ALLOW_ALL` from
  `getTargetRestriction`, so they can target constructors without `unsafe`
  [BeforeReturn.java:getTargetRestriction; BeforeFinalReturn.java:getTargetRestriction].
- `TAIL` throws `InvalidInjectionException("TAIL could not locate a valid RETURN in the target method!")`
  when no matching return exists [BeforeFinalReturn.java:find].
- `CTOR_HEAD`:
  - It requires `unsafe = true`. Otherwise the constructor throws
    `InvalidInjectionPointException("@At(\"CTOR_HEAD\") requires unsafe=true")`.
  - The `enforce` argument takes `DEFAULT`, `POST_DELEGATE`, `POST_INIT` or
    `PRE_BODY`, and defaults to `DEFAULT`.
  - It returns `false` unless `insns` is an `IInsnListEx`.
  - On a non-constructor it behaves like `HEAD`.
  - On a constructor, `POST_DELEGATE` selects the node after the delegate
    call. Without a delegate it returns `false` and logs a WARN, only under
    `DEBUG_VERBOSE`.
  - Otherwise it tries the `CTOR_BODY` special node for `PRE_BODY`, or the
    `INITIALISER_INJECTION_POINT` node for the other modes. It then falls back
    to the node after the delegate call, and finally to `HEAD`.
  [ConstructorHead.java:ConstructorHead; find]
- `INVOKE` matching:
  - The selector is configured with `Configure.SELECT_INSTRUCTION`, and only
    `isExactMatch()` results count.
  - Matching stops once more than `getMaxMatchCount()` matches are seen.
  - Fewer than `getMinMatchCount()` matches throw
    `SelectorConstraintException("%s did not match the required number of targets (required=%d, matched=%d)")`.
  [BeforeInvoke.java:find(String,InsnList,Collection,ITargetSelector,SearchType)]
- Permissive search is enabled only when all three of these hold:
  `REFMAP_REMAP` is on, `REFMAP_REMAP_ALLOW_PERMISSIVE` is on, and the
  reference mapper is not the default. It runs only when the strict search
  found nothing and the target is an `ITargetSelectorByName` with a `null`
  descriptor. Both stages log a WARN [BeforeInvoke.java:BeforeInvoke; find].
- `log = true` in the arguments makes `BeforeInvoke` and its subclasses log
  the search at INFO. `BeforeConstant` does the same with `log`
  [BeforeInvoke.java:log; BeforeConstant.java:log].
- `INVOKE_ASSIGN`:
  - It skips void calls.
  - `fuzz` is at least 1 and defaults to 1. Within that window, a following
    `VarInsnNode` moves the point to that store.
  - The search stops at the first opcode not in `skip`. `skip` defaults to
    `DEFAULT_SKIP` and is parsed by `getOpcodeList`.
  - The selected node is the instruction after the chosen one.
  [AfterInvoke.java:AfterInvoke; addInsn]
- `INVOKE_STRING` needs two things:
  - the `ldc` argument. Without it the constructor throws
    `IllegalArgumentException("BeforeStringInvoke requires named argument \"ldc\" ...")`;
  - a target selector whose descriptor is exactly `(Ljava/lang/String;)V`.
    Otherwise it throws `IllegalArgumentException(... "requires target method with with signature (Ljava/lang/String;)V")`.

  `InjectionPoint.create` wraps either exception in `InvalidInjectionException`
  [BeforeStringInvoke.java:BeforeStringInvoke; InjectionPoint.java:create].
- `FIELD`:
  - The opcode is accepted only when it is `GETFIELD`, `PUTFIELD`, `GETSTATIC`
    or `PUTSTATIC`. Anything else, including the default 0, means any field
    opcode.
  - `array` takes `get`, `set` or `length`, case-insensitively. With `array`
    set, only array-typed `GET*` instructions match, and only when
    `findArrayNode` finds the access within `fuzz`. `fuzz` is clamped to 1 to
    32 and defaults to 8.
  [BeforeFieldAccess.java:BeforeFieldAccess; matchesInsn; addInsn]
- `findArrayNode` returns `null` in three cases: it meets an `ARRAYLENGTH` at
  position 0, it meets another access to the same field, or it passes the
  search range [BeforeFieldAccess.java:findArrayNode].
- `NEW`:
  - It reads the `class` argument, falling back to `target`. The value is
    parsed as a selector and must be an `ITargetSelectorConstructor`, or the
    constructor throws `InvalidInjectionPointException("Failed parsing @At(\"NEW\") target descriptor ...")`.
  - With a descriptor, a `NEW` is selected only when `findInitNodeFor` finds
    the matching `<init>` at the same nesting depth.
  [BeforeNew.java:BeforeNew; find; findInitNodeFor]
- `JUMP` filters by `opcode` only when it is one of the jump opcodes (`IFEQ`
  to `IF_ACMPNE`, `GOTO`, `JSR`, `IFNULL`, `IFNONNULL`). It is built with the
  no-argument `InjectionPoint()` constructor, so it ignores the `@At` `slice`,
  specifier and `id`, and it has no message sink [JumpInsnPoint.java:JumpInsnPoint].
- `CONSTANT` discriminators:
  - At most one of `nullValue`, `intValue`, `floatValue`, `longValue`,
    `doubleValue`, `stringValue` and `classValue` may be set. Setting more
    throws `InvalidInjectionException("Conflicting constant discriminators specified ...")`.
  - In `@At("CONSTANT")` arguments, setting none throws
    `InvalidInjectionException("No constant discriminator could be parsed in @At(\"CONSTANT\") args")`.
  - With the `@Constant` constructor, setting none matches every constant of
    the handler's return type.
  [BeforeConstant.java:validateDiscriminator; BeforeConstant(InjectionPointData)]
- `CONSTANT` with `expandZeroConditions` matches the conditional jump opcodes
  of the listed `Constant.Condition`s. It ignores a jump that directly follows
  `LCMP`, `FCMPL`, `FCMPG`, `DCMPL` or `DCMPG` [BeforeConstant.java:matchesConditionalInsn].

## Side Effects

- `BeforeInvoke`, `BeforeConstant` and `ConstructorHead` get the `"mixin"`
  logger from `MixinService` [BeforeInvoke.java:logger; BeforeConstant.java:logger].
- `find` adds to the caller's `nodes` collection and changes no instructions.
  `BeforeStringInvoke` resets its per-search `foundLdc` state on each `find`
  [BeforeStringInvoke.java:find].

## Verification

- Compile: `mise exec -- ./gradlew compileJava` (run from the repository root).
- Test: `mise exec -- ./gradlew test`. `build.gradle` declares the `test`
  source set, but `src/test` does not exist yet, so there are no tests today.
