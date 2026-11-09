# invoke

## Overview

Injectors that work on one instruction inside a target method:

- `@Redirect` replaces a method call, field access, constructor call or
  `instanceof`.
- `@ModifyArg` changes one argument of a call.
- `@ModifyArgs` changes all of a call's arguments through a generated `Args`
  bundle.
- `@ModifyConstant` replaces a constant load or a zero-comparison jump.

Each extends `Injector` from
src/main/java/org/spongepowered/asm/mixin/injection/code/Injector.java.

## Interface

- `public abstract class InvokeInjector extends Injector` [InvokeInjector.java:InvokeInjector]
  - `InvokeInjector(InjectionInfo, String annotationType)`.
  - `protected void checkTarget(Target)` calls `checkTargetModifiers(target, true)`
    by default.
  - `protected abstract void injectAtInvoke(Target, InjectionNode)`.
- `public class RedirectInjector extends InvokeInjector` [RedirectInjector.java:RedirectInjector]
  - Constructors: the public `RedirectInjector(InjectionInfo)` and the
    protected `(InjectionInfo, String annotationType)`.
  - Protected members `meta`, `preInject(InjectionNode)`,
    `injectAtConstructor`, `injectAtInstanceOf(Target, InjectionNode)` and
    `injectAtInstanceOf(Target, TypeInsnNode)`.
  - `Meta.KEY = "redirector"` and `ConstructorRedirectData.KEY = "ctor"` are
    node decoration keys.
- `public class ModifyArgInjector extends InvokeInjector` with the constructor
  `(InjectionInfo, int index)` and the protected method
  `int findArgIndex(Target, Type[])` [ModifyArgInjector.java:ModifyArgInjector].
- `public class ModifyArgsInjector extends InvokeInjector` with the constructor
  `(InjectionInfo)`. It gets its `ArgsClassGenerator` from
  `info.getMixin().getExtensions()` [ModifyArgsInjector.java:ModifyArgsInjector].
- `public class ModifyConstantInjector extends RedirectInjector` with the
  constructor `(InjectionInfo)` [ModifyConstantInjector.java:ModifyConstantInjector].
- Subdirectories:
  - [arg](./arg/EGM.md): `Args`, `ArgsClassGenerator` (synthetic
    `org.spongepowered.asm.synthetic.args.Args$N` classes),
    `ArgumentCountException` and `ArgumentIndexOutOfBoundsException`.
  - [util](./util/EGM.md): `InsnFinder.findPopInsn`, which finds the
    instruction that consumes a pushed value.

## Invariants

- `InvokeInjector.inject` throws
  `InvalidInjectionException("%s annotation on is targetting a non-method insn in %s in %s")`
  when the current target is not a `MethodInsnNode`. `RedirectInjector` and
  `ModifyConstantInjector` override `inject` and do not use this check
  [InvokeInjector.java:inject].
- Static rules:
  - `ModifyArgInjector.checkTarget` rejects only a non-static handler on a
    static target.
  - `ModifyArgsInjector` uses `checkTargetModifiers(target, false)`.
  - `RedirectInjector.checkTarget` does nothing. Each node is checked later
    with `checkTargetForNode(..., ALLOW_ALL)`.
  [ModifyArgInjector.java:checkTarget; ModifyArgsInjector.java:checkTarget; RedirectInjector.java:checkTarget]
- Redirect conflicts are decided per instruction through the `"redirector"`
  decoration. Suppose another redirector already owns the node:
  - If its priority is at least this mixin's priority, this injector logs a
    WARN `"%s conflict. Skipping ..."` and skips the node.
  - Otherwise, if the owner is `@Final`, it throws
    `InvalidInjectionException("%s conflict: %s failed because target was already remapped by %s")`.
  - Otherwise this injector takes the node over.

  `preInject` skips nodes that another redirector owns at inject time
  [RedirectInjector.java:addTargetNode; preInject].
- Redirecting a `<init>` `MethodInsnNode` throws
  `InvalidInjectionException("Illegal %s of constructor specified on %s")`. Use
  `@At("NEW")` instead [RedirectInjector.java:addTargetNode].
- `inject` on a node that has been replaced throws
  `UnsupportedOperationException("Redirector target failure for ...")`.
  `ModifyConstantInjector` throws `"Target failure for ..."` instead
  [RedirectInjector.java:inject; ModifyConstantInjector.java:inject].
- Method redirects:
  - The handler arguments are the receiver (unless the call is
    `INVOKESTATIC`) followed by the call's arguments, then optionally a prefix
    of the target method's arguments.
  - With a coerced return type, an object or array result gets a `CHECKCAST`.
  - The node is decorated with `ArgOffsets(isStatic ? 0 : 1, argCount)`.
  [RedirectInjector.java:RedirectedInvokeData; injectAtInvoke]
- Field redirects:
  - A getter handler takes `(owner)` for an instance field and returns the
    field type.
  - A setter handler takes `(owner, value)` and returns `void`.
  - A handler with more array dimensions than the field throws
    `"Dimensionality of handler method is greater than target array ..."`.
  - Element access on an array field is allowed only for `GETSTATIC` and
    `GETFIELD`. Other opcodes throw `"Unspported opcode %s for array access %s"`.
  - When the array instruction cannot be found, it throws
    `"Array element %s on %s could not locate a matching %s instruction in %s."`.
  [RedirectInjector.java:injectAtFieldAccess; injectAtArrayField; injectArrayRedirect]
- Constructor redirects (`NEW`):
  - A non-static handler with a static target throws.
  - The handler is a factory that returns the constructed type.
  - When the `NEW` is followed by a `DUP`, a null result throws a
    `NullPointerException` at runtime with the message
    `"%s constructor handler %s returned null for %s"`. Otherwise the result is
    popped.
  - When `NEW` has no descriptor (wildcard), lookup and validation failures are
    collected instead of thrown. In that case `postInject` throws
    `"%s ctor invocation was not found in %s"` if nothing was injected.
  [RedirectInjector.java:injectAtConstructor; ConstructorRedirectData.throwOrCollect; postInject]
- `instanceof` redirects:
  - The handler must return `boolean` or `Class`, and must take
    `(Object, Class)` plus optional target arguments, without coercion.
    Anything else throws `"... INSTANCEOF handler expects ..."`.
  - With a `Class` return, the emitted code checks for null and then calls
    `Class.isAssignableFrom`. A null object yields `false`.
  [RedirectInjector.java:injectAtInstanceOf; redirectInstanceOf; checkIsAssignableFrom]
- `ModifyArgInjector`:
  - In single-argument mode (the handler takes one argument), the handler's
    return type must equal its parameter type, or `sanityCheck` throws.
  - In multi-argument mode, the handler arguments must equal the call's
    arguments. Otherwise it throws `"... targets a method with an invalid signature ..."`.
  - A call with no arguments throws `"... with no arguments!"`.
  [ModifyArgInjector.java:sanityCheck; injectAtInvoke]
- `ModifyArgInjector.findArgIndex`:
  - An explicit `index` must be within range and of the return type, or it
    throws `"Specified index %d for @ModifyArg is invalid ..."`.
  - Without one, exactly one argument may have the return type. Two throw
    `"Found duplicate args with index [a, b] ..."` and none throws
    `"Could not find arg matching type ..."`.
  [ModifyArgInjector.java:findArgIndex]
- `ArgOffsets` decorations left by an earlier `@Redirect` move the argument
  window that `@ModifyArg` and `@ModifyArgs` see, so they can target a call
  that has already been redirected [ModifyArgInjector.java:injectAtInvoke; ModifyArgsInjector.java:injectAtInvoke].
- `ModifyArgsInjector` handler descriptors:
  - It must be `(Lorg/spongepowered/asm/mixin/injection/invoke/arg/Args;)V`,
    or that followed by the target method's arguments with a `V` return.
    Otherwise it throws `"@ModifyArgs injector ... has an invalid signature ..., expected ... or ..."`.
  - A call with no arguments throws `"... with no arguments!"`.
  [ModifyArgsInjector.java:verifyTarget; injectAtInvoke]
- `ModifyConstantInjector` accepts these target instructions:
  - a constant instruction, which is wrapped;
  - an `INSTANCEOF`, handled as a redirect. Other `TypeInsnNode` opcodes throw
    `"does not support %s insn"`;
  - a `JumpInsnNode` from `IFLT` to `IFLE`. Other jump opcodes throw
    `"selected an invalid opcode"`. The jump is replaced with `ICONST_0`,
    the handler call, and the matching `IF_ICMPxx`.
  - Any other instruction throws `"%s annotation is targetting an invalid insn ..."`.
  [ModifyConstantInjector.java:inject; injectExpandedConstantModifier; injectTypeConstantModifier]
- A `@ModifyConstant` handler must take the constant type and return the same
  type, optionally followed by target arguments [ModifyConstantInjector.java:invokeConstantHandler].

## Side Effects

- These injectors rewrite target instruction lists through
  `Target.replaceNode`, `wrapNode`, `removeNode` and `insertBefore`. They also
  extend max stack and max locals [RedirectInjector.java:injectAtInvoke; ModifyConstantInjector.java:injectConstantModifier].
- They add `"redirector"`, `"nominators"`, `"ctor"`, `"fuzz"`, `"opcode"` and
  `ArgOffsets.KEY` decorations to injection nodes [RedirectInjector.java:addTargetNode; injectAtInvoke].
- `ModifyArgsInjector` asks `ArgsClassGenerator.getArgsClass` for a synthetic
  class, which registers it on first use. The emitted code calls
  `Args$N.of(...)` and the `$N` getters [ModifyArgsInjector.java:injectAtInvoke; packArgs; unpackArgs].
- Logging through the `"mixin"` logger:
  - Redirect conflicts log at WARN.
  - Under `DEBUG_VERBOSE`, `@ModifyConstant` checks narrowing conversions with
    `InsnFinder`. It logs at WARN, or at ERROR when the conversion is to
    `boolean`.
  [RedirectInjector.java:preInject; ModifyConstantInjector.java:checkNarrowing]

## Verification

- Compile: `mise exec -- ./gradlew compileJava` (run from the repository root).
- Test: `mise exec -- ./gradlew test`. `build.gradle` declares the `test`
  source set, but `src/test` does not exist yet, so there are no tests today.
