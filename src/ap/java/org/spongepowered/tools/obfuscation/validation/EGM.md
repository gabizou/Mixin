# validation

## Overview

Checks that run on each mixin class. Both validators extend `MixinValidator`
from the parent package. `AnnotatedMixins` creates them in this order:
`ParentValidator`, then `TargetValidator`. Each runs only during its own
`ValidationPass`.

## Interface

- `ParentValidator`: public class that extends `MixinValidator`. The
  constructor is `(IMixinAnnotationProcessor)` and its pass is `EARLY`
  [ParentValidator.java:ParentValidator].
- `TargetValidator`: public class that extends `MixinValidator`. The
  constructor is `(IMixinAnnotationProcessor)` and its pass is `LATE`
  [TargetValidator.java:TargetValidator].
- Both override `validate(TypeElement mixin, IAnnotationHandle annotation, Collection<TypeHandle> targets)`
  and always return `true`, so later validators still run
  [ParentValidator.java:validate].

## Invariants

### ParentValidator (EARLY pass, when the mixin is registered)

- If the mixin's enclosing element is not a package and the mixin is not
  `static`, it emits `PARENT_VALIDATOR` (default `ERROR`) with the text
  `Inner class mixin must be declared static`, positioned on the mixin
  element [ParentValidator.java:validate].

### TargetValidator (LATE pass, at the end of the round that registered the mixin)

- It is skipped completely when the option `disableTargetValidator` equals
  `true`, ignoring case [TargetValidator.java:validate].
- Interface mixins: the check runs only if the mixin declares at least one
  method without `@Accessor` or `@Invoker`. In that case, each non-null target
  where `isNotInterface()` is true gets `TARGET_VALIDATOR` (default `ERROR`)
  with the text `Targetted type '<target> of <mixin> is not an interface`.
  The quote mark is unbalanced in the source text [TargetValidator.java:validateInterfaceMixin].
- Class mixins: the mixin's superclass is resolved through the type handle
  provider. A non-null target passes if any of these is true. Otherwise it
  gets `TARGET_VALIDATOR` with the text
  `Superclass <super> of <mixin> was not found in the hierarchy of target class <target>`
  [TargetValidator.java:validateClassMixin].
  - The target is imaginary.
  - The target is simulated.
  - The superclass is a supertype of the target.
  - Any `@Mixin` `value` class or `targets` string on the superclass is a
    supertype of the target [TargetValidator.java:checkMixinsFor].
- Both messages are positioned on the mixin element. They go through
  `IMessagerSuppressible`, so `MSG_PARENT_VALIDATOR` and
  `MSG_TARGET_VALIDATOR` can change their kind or disable them
  [TargetValidator.java:validateClassMixin].
- If an IDE is detected and no obfuscation environment is available,
  `ObfuscationManager.init` lowers both message types from `ERROR` to
  `WARNING`. See
  `src/ap/java/org/spongepowered/tools/obfuscation/ObfuscationManager.java`.

## Side Effects

- These validators write compiler diagnostics. They change no other state
  [TargetValidator.java:validate].

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileApJava`
- Run: `mise exec -- ./gradlew test`. Note that `src/test` does not exist yet,
  so no tests cover this package today.
