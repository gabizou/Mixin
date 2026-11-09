# modify

## Overview

The `@ModifyVariable` injector and the local-variable injection points it
uses. `ModifyVariableInjector` loads a chosen local, passes it through the
handler, and stores the result back. `LocalVariableDiscriminator` picks the
local by ordinal, by index, by name, or implicitly as the only local of the
handler's type. `BeforeLoadLocal` (`LOAD`) and `AfterStoreLocal` (`STORE`) find
the loads and stores of that local. The injector is configured by
`ModifyVariableInjectionInfo` in
src/main/java/org/spongepowered/asm/mixin/injection/struct/ModifyVariableInjectionInfo.java.

## Interface

- `public class ModifyVariableInjector extends Injector` with the constructor
  `(InjectionInfo, LocalVariableDiscriminator)` [ModifyVariableInjector.java:ModifyVariableInjector].
  - The nested `abstract static class LocalVariableInjectionPoint extends InjectionPoint`
    is package-private [ModifyVariableInjector.java:LocalVariableInjectionPoint].
  - The nested `static class Context extends LocalVariableDiscriminator.Context`
    holds the instructions it generates [ModifyVariableInjector.java:Context].
- `@AtCode("LOAD") public class BeforeLoadLocal extends LocalVariableInjectionPoint`.
  Its constructors are protected. `toString()` is
  `@At("<code>" ordinal=%d index=%d)` [BeforeLoadLocal.java:BeforeLoadLocal].
- `@AtCode("STORE") public class AfterStoreLocal extends BeforeLoadLocal` with
  the public constructor `(InjectionPointData)` [AfterStoreLocal.java:AfterStoreLocal].
- `public class LocalVariableDiscriminator` [LocalVariableDiscriminator.java:LocalVariableDiscriminator]
  - `LocalVariableDiscriminator(boolean argsOnly, int ordinal, int index, Set<String> names, boolean print)`.
  - `static parse(AnnotationNode)`.
  - Accessors: `isArgsOnly`, `getOrdinal`, `getIndex`, `getNames`, `hasNames`
    and `printLVT`.
  - `int findLocal(Context)` and `toString(Context)`.
  - The nested `public static class Context implements PrettyPrinter.IPrettyPrintable`
    provides `getCandidateCount()` and `print(PrettyPrinter)`. Its nested
    `Local` class holds the name, type and ordinal.
- `public class InvalidImplicitDiscriminatorException extends MixinException`
  [InvalidImplicitDiscriminatorException.java:InvalidImplicitDiscriminatorException].

## Invariants

- `LocalVariableDiscriminator.parse` reads `argsOnly` (default `false`),
  `ordinal` (default -1), `index` (default -1), `print` (default `false`) and
  `name` (default none) [LocalVariableDiscriminator.java:parse].
- A discriminator is implicit when `ordinal < 0`, `index < baseArgIndex` and no
  names are set. `baseArgIndex` is 0 for a static target and 1 otherwise
  [LocalVariableDiscriminator.java:isImplicit].
- Implicit mode returns the only local, at or after `baseArgIndex`, whose type
  equals the handler return type. A count other than 1 throws
  `InvalidImplicitDiscriminatorException("Found <n> candidate variables but exactly 1 is required.")`
  [LocalVariableDiscriminator.java:findImplicitLocal].
- Explicit mode looks only at locals of the handler return type, in this order:
  1. When `ordinal > -1`, it matches the per-type ordinal.
  2. Otherwise, when `index >= baseArgIndex`, it matches the slot index.
  3. Otherwise it matches by name.

  It returns -1 when nothing matches [LocalVariableDiscriminator.java:findExplicitLocal].
- Building a `Context`:
  - When `argsOnly` is false and `Locals.getLocalsAt` finds locals, those are
    used.
  - Otherwise the context is built from the target arguments. Slot 0 is
    `"this"` for an instance method, the rest are named `"arg" + slot`, and
    wide types leave an empty following slot.
  - Ordinals count from 0 for each distinct type.
  [LocalVariableDiscriminator.java:Context.initLocals; initOrdinals]
- Setting a different ordinal on a `Local` that already has one throws
  `IllegalStateException("Attempted to reset ordinal for computed local")`
  [LocalVariableDiscriminator.java:Context.Local.setOrdinal].
- `sanityCheck` throws `InvalidInjectionException` in two cases:
  - an ordinal below -1: `"Invalid ordinal <n> specified in ..."`;
  - index 0 on a non-static target: `"Invalid index 0 specified in non-static variable modifier ..."`.
  [ModifyVariableInjector.java:sanityCheck]
- Errors raised in `inject`:
  - A node that was replaced throws `InvalidInjectionException("Variable modifier target for ... was removed by another injector")`.
  - A missing `CONTEXT` decoration throws `InjectionError`, and so does one
    that already contains instructions.
  - A `void` handler return type throws `InvalidInjectionException("... cannot return a VOID type.")`.
  [ModifyVariableInjector.java:inject]
- The handler is validated with `allowCoerceArgs = false`. The handler takes
  the variable type and may then take a prefix of the target's arguments
  [ModifyVariableInjector.java:inject]. `validateParams` is in
  src/main/java/org/spongepowered/asm/mixin/injection/code/Injector.java.
- An implicit discriminator failure throws
  `InvalidInjectionException("Implicit variable modifier injection failed in ...", cause)`.
  When `print` is set, the failure is suppressed instead
  [ModifyVariableInjector.java:inject].
- When `findLocal` returns -1 (no explicit match), nothing is injected for that
  node [ModifyVariableInjector.java:inject].
- `LocalVariableInjectionPoint.find(desc, insns, nodes)` needs an `IInsnListEx`
  decorated with `mv.target`. Otherwise it throws
  `InvalidInjectionException("<code> injection point must be used in conjunction with @ModifyVariable")`
  [ModifyVariableInjector.java:LocalVariableInjectionPoint.find].
- `findTargetNodes` adds the `mv.target` and `mv.info` decorations to the slice
  before the search and removes them afterwards
  [ModifyVariableInjector.java:findTargetNodes].
- `LOAD` matches `VarInsnNode`s with the opcode
  `data.getOpcode(returnType.getOpcode(ILOAD))`. `STORE` does the same with
  `ISTORE`, but checks on the next instruction, so the selected node is the one
  after the store [BeforeLoadLocal.java:BeforeLoadLocal(InjectionPointData,int,boolean); find].
- The `ordinal` counts only accesses to the discriminated local. When an
  ordinal is set, the search stops matching after the first success
  [BeforeLoadLocal.java:find; SearchState.check].
- A context node key is
  `"localcontext(<returnType>,argsOnly|fullFrame,#<nodeId>)"`. One context is
  computed per key and reused [ModifyVariableInjector.java:getTargetNodeKey; preInject].

## Side Effects

- Inserts `[ALOAD 0,] xLOAD local, [target args,] INVOKE handler, xSTORE local`
  before the target node. Extends max stack and records the callback call
  [ModifyVariableInjector.java:inject(Context,InjectorData,Extension,int)].
- When `print` is set, the injector prints a local-variable table to
  `System.err` [ModifyVariableInjector.java:printLocals].
- During the search, an invalid implicit discriminator is recorded as an
  injection point message through `addMessage(...)`, not thrown
  [BeforeLoadLocal.java:SearchState.check].
- Adds and removes decorations on `InjectionNode`s and on the slice
  `InsnListEx` [ModifyVariableInjector.java:preInject; findTargetNodes].

## Verification

- Compile: `mise exec -- ./gradlew compileJava` (run from the repository root).
- Test: `mise exec -- ./gradlew test`. `build.gradle` declares the `test`
  source set, but `src/test` does not exist yet, so there are no tests today.
