# callback

## Overview

Runtime and injection-time support for `@Inject`. `CallbackInfo` and
`CallbackInfoReturnable` are the objects handlers receive at runtime, and they
carry cancellation and return values. `LocalCapture` selects the local-capture
mode. `CallbackInjector` is the `Injector` that writes the handler call, the
`CallbackInfo` creation, and the cancellation and return code into target
methods. Its configuration comes from `CallbackInjectionInfo` in
src/main/java/org/spongepowered/asm/mixin/injection/struct/CallbackInjectionInfo.java.

## Interface

- `public interface Cancellable` declares `isCancellable()`, `isCancelled()`
  and `cancel() throws CancellationException` [Cancellable.java:Cancellable].
- `public class CancellationException extends RuntimeException`. Its
  constructors are `()`, `(String)`, `(Throwable)` and `(String, Throwable)`
  [CancellationException.java:CancellationException].
- `public class CallbackInfo implements Cancellable` [CallbackInfo.java:CallbackInfo]
  - `CallbackInfo(String name, boolean cancellable)`.
  - `String getId()` returns `name`.
  - `final boolean isCancellable()` and `final boolean isCancelled()`.
  - `void cancel()`.
  - `toString()` returns `"CallbackInfo[TYPE=%s,NAME=%s,CANCELLABLE=%s]"`, where
    TYPE is the class's simple name.
  - `static String getCallInfoClassName(Type returnType)` returns the internal
    name of `CallbackInfo` for `void` and of `CallbackInfoReturnable` otherwise.
- `public class CallbackInfoReturnable<R> extends CallbackInfo`
  [CallbackInfoReturnable.java:CallbackInfoReturnable]
  - Constructors: `(String, boolean)`, `(String, boolean, R)`, and one overload
    for each primitive (`byte`, `char`, `double`, `float`, `int`, `long`,
    `short`, `boolean`) that boxes the value.
  - `void setReturnValue(R)` and `R getReturnValue()`.
  - Primitive getters `getReturnValueB`, `getReturnValueC`, `getReturnValueD`,
    `getReturnValueF`, `getReturnValueI`, `getReturnValueJ`, `getReturnValueS`
    and `getReturnValueZ`.
- `public enum LocalCapture` has the constants `NO_CAPTURE`, `PRINT`,
  `CAPTURE_FAILSOFT`, `CAPTURE_FAILHARD` and `CAPTURE_FAILEXCEPTION`
  [LocalCapture.java:LocalCapture].
- `public class CallbackInjector extends Injector`
  [CallbackInjector.java:CallbackInjector]
  - `CallbackInjector(InjectionInfo info, boolean cancellable, LocalCapture localCapture, String identifier)`.
  - The protected hooks `instanceCallbackInfo`, `injectCancellationCode`,
    `injectReturnCode` and `isStatic()` can be overridden.

## Invariants

- `CallbackInfo.cancel()` throws
  `CancellationException("The call " + name + " is not cancellable.")` when the
  instance is not cancellable. Otherwise it sets `isCancelled()` to `true`
  [CallbackInfo.java:cancel].
- `CallbackInfoReturnable.setReturnValue` calls `super.cancel()` before it
  stores the value. Setting a return value therefore cancels the call, and on a
  non-cancellable instance it throws `CancellationException` without changing
  the stored value [CallbackInfoReturnable.java:setReturnValue].
- When the stored value is `null`, each primitive getter returns its zero value
  (`0`, `0.0`, `0.0F` or `false`). Otherwise it unboxes with a cast, so a value
  of the wrong boxed type throws `ClassCastException`
  [CallbackInfoReturnable.java:getReturnValueI].
- `CallbackInfoReturnable.getReturnAccessor` maps object and array returns to
  `"getReturnValue"` and maps a primitive to `"getReturnValue" + descriptor`.
  For example, `I` maps to `getReturnValueI`
  [CallbackInfoReturnable.java:getReturnAccessor].
- The `CallbackInfo` constructor descriptor depends on the return type:
  - `(Ljava/lang/String;Z)V` for `void`;
  - `(Ljava/lang/String;ZLjava/lang/Object;)V` for object and array types;
  - `(Ljava/lang/String;Z<prim>)V` for primitives
  [CallbackInfo.java:getConstructorDescriptor(Type)].
- `LocalCapture` flags: `NO_CAPTURE` neither captures nor prints. `PRINT` only
  prints. The three `CAPTURE_*` constants capture and do not print
  [LocalCapture.java:LocalCapture()].
- `sanityCheck` calls `checkTargetModifiers(target, false)`. A non-static
  handler that targets a static method throws `InvalidInjectionException`
  [CallbackInjector.java:sanityCheck].
- A cancellable `@Inject` whose target is a `Constructor` throws
  `InvalidInjectionException("Found cancellable @Inject targetting a constructor in injector %s")`
  [CallbackInjector.java:addTargetNode].
- Errors from `checkTargetForNode` are rethrown as `InvalidInjectionException`
  with the message `"<injection point> selector <original message>"`
  [CallbackInjector.java:addTargetNode].
- When two injection points give the same instruction different ids, the
  injector logs a WARN `"Conflicting id ..."` and keeps the first id
  [CallbackInjector.java:addTargetNode].
- Local capture removes any local whose descriptor starts with
  `Lorg/spongepowered/asm/mixin/injection/callback/`. Capture is only possible
  when the local table is larger than the argument frame
  [CallbackInjector.java:preInject; CallbackInjector.java:Callback].
- Descriptor matching tries these rules in order:
  1. An exact match with the expected descriptor is accepted.
  2. The simple descriptor without locals is accepted when locals cannot be
     captured. Arguments are then not passed.
  3. Otherwise each argument type must match. A differing argument passes only
     when it is not an array, has `@Coerce`, and `Injector.canCoerce` allows it.
  [CallbackInjector.java:Callback.checkDescriptor]
- When the descriptor does not match and the injector has more than one
  target, the current target is skipped silently
  [CallbackInjector.java:inject(Callback)].
- When the descriptor does not match and locals can be captured:
  - A same-named `@Surrogate` method with the expected descriptor is used if
    one exists.
  - Otherwise `CAPTURE_FAILEXCEPTION` logs an ERROR and generates
    `<handler>$missing`, which throws `InjectionError`.
  - `CAPTURE_FAILSOFT` logs a WARN and skips the target.
  - Any other mode logs an ERROR and throws `InjectionError(message)`.
  [CallbackInjector.java:inject(Callback)]
- When the descriptor does not match and locals cannot be captured:
  - If the handler would match with `CallbackInfoReturnable` in place of
    `CallbackInfo`, it throws
    `InvalidInjectionException("Invalid descriptor on <info>! CallbackInfoReturnable is required!")`.
  - Otherwise a same-named `@Surrogate` method is used if one exists.
  - If neither applies, it throws
    `InvalidInjectionException("Invalid descriptor on <info>! Expected <desc> but found <desc>")`.
  [CallbackInjector.java:inject(Callback)]
- The `CallbackInfo` id is the `identifier` value, or the target method name
  when that is empty. When the injection point has an id, `":" + id` is
  appended [CallbackInjector.java:getIdentifier].
- The `CallbackInfo` is stored in a local variable when the injector is
  cancellable or has more than one injection. In that case the variable is
  named `"callbackInfo" + var`
  [CallbackInjector.java:inject(Callback); instanceCallbackInfo].
- A non-cancellable injector that is not at a return can reuse the previous
  `CallbackInfo` when the id and constructor descriptor are unchanged in the
  same target [CallbackInjector.java:createCallbackInfo].
- Cancellation code is emitted only for cancellable injectors. It tests
  `isCancelled()` and, when true, returns. A `void` target returns directly.
  Otherwise the value comes from the typed `getReturnValue*` accessor, with a
  `CHECKCAST` for object and array types
  [CallbackInjector.java:injectCancellationCode; injectReturnCode].
- At a value-returning instruction (opcodes `IRETURN` up to but not including
  `RETURN`), the return value is duplicated and stored, then passed to the
  `CallbackInfoReturnable` constructor [CallbackInjector.java:dupReturnValue; Callback.isAtReturn].
- Local names that begin with the Unicode snowman character are shown as
  `"var" + index` [CallbackInjector.java:meltSnowman].

## Side Effects

- Modifies the target method's instruction list. It may allocate locals with
  `target.allocateLocal()` and extends max stack and max locals
  [CallbackInjector.java:Callback.inject; Callback.marshalVar].
- Adds a local variable entry for the stored `CallbackInfo`
  [CallbackInjector.java:instanceCallbackInfo].
- `CAPTURE_FAILEXCEPTION` adds a synthetic method `<handler>$missing` to the
  target class through `info.addMethod` [CallbackInjector.java:generateErrorMethod].
- `LocalCapture.PRINT` prints the target's locals and the expected callback
  signature to `System.err` and injects nothing. It still calls
  `info.addCallbackInvocation` [CallbackInjector.java:printLocals; inject(Callback)].
- Logs through `Injector.logger` (the `"mixin"` logger):
  - WARN for id conflicts and `CAPTURE_FAILSOFT` failures;
  - ERROR for `CAPTURE_FAILEXCEPTION` and hard local-capture failures
  [CallbackInjector.java:inject(Callback)].
- Each successful injection calls `info.notifyInjected(target)`
  [CallbackInjector.java:inject(Callback)].

## Verification

- Compile: `mise exec -- ./gradlew compileJava` (run from the repository root).
- Test: `mise exec -- ./gradlew test`. `build.gradle` declares the `test`
  source set, but `src/test` does not exist yet, so there are no tests today.
