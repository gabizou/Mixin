# invoke/arg

## Overview

Runtime argument bundles for `@ModifyArgs` handlers. `Args` is the abstract
bundle that handlers receive. `ArgsClassGenerator` assigns one synthetic `Args`
subclass to each distinct argument list and writes its bytecode on demand. The
generated subclass has a private constructor, a `toString`, a static `of`
factory, typed `$N` getters, and checked `set` and `setAll` methods. The
consumer is `ModifyArgsInjector` in
src/main/java/org/spongepowered/asm/mixin/injection/invoke/ModifyArgsInjector.java.

## Interface

- `public abstract class Args` [Args.java:Args]
  - `protected final Object[] values`. The protected constructor stores the
    array reference it is given and does not copy it [Args.java:Args(Object[])].
  - `int size()` returns `values.length` [Args.java:size].
  - `<T> T get(int index)` returns `(T) values[index]` with an unchecked cast
    [Args.java:get].
  - `abstract <T> void set(int index, T value)` and
    `abstract void setAll(Object... values)` [Args.java:set; Args.java:setAll].
- `public final class ArgsClassGenerator implements IClassGenerator`. The
  supertype is in
  src/main/java/org/spongepowered/asm/mixin/transformer/ext/IClassGenerator.java
  [ArgsClassGenerator.java:ArgsClassGenerator].
  - Constants: `ARGS_NAME` is
    `"org.spongepowered.asm.mixin.injection.invoke.arg.Args"`, `ARGS_REF` is
    the same name with `/` separators, `GETTER_PREFIX` is `"$"`, and
    `SYNTHETIC_PACKAGE` is `"org.spongepowered.asm.synthetic.args"`
    [ArgsClassGenerator.java:ARGS_NAME; ARGS_REF; GETTER_PREFIX; SYNTHETIC_PACKAGE].
  - `ArgsClassGenerator(IConsumer<ISyntheticClassInfo> registry)`
    [ArgsClassGenerator.java:ArgsClassGenerator(IConsumer)].
  - `String getName()` returns `"args"` [ArgsClassGenerator.java:getName].
  - `ISyntheticClassInfo getArgsClass(String desc, IMixinInfo mixin)` assigns a
    class name or returns the existing one. It does not generate bytecode
    [ArgsClassGenerator.java:getArgsClass].
  - `boolean generate(String name, ClassNode classNode)` writes the class into
    `classNode` [ArgsClassGenerator.java:generate].
- `public class ArgumentCountException extends IllegalArgumentException` with
  the constructor `(int received, int expected, String desc)`
  [ArgumentCountException.java:ArgumentCountException].
- `public class ArgumentIndexOutOfBoundsException extends IndexOutOfBoundsException`
  with the constructor `(int index)`
  [ArgumentIndexOutOfBoundsException.java:ArgumentIndexOutOfBoundsException].

## Invariants

- The `ArgumentCountException` message is exactly
  `"Invalid number of arguments for setAll, received " + received + " but expected " + expected + ": " + desc`
  [ArgumentCountException.java:ArgumentCountException].
- The `ArgumentIndexOutOfBoundsException` message is exactly
  `"Argument index is out of bounds: " + index`
  [ArgumentIndexOutOfBoundsException.java:ArgumentIndexOutOfBoundsException].
- `Args.get` does no bounds check of its own. An out-of-range index throws
  `ArrayIndexOutOfBoundsException`, not `ArgumentIndexOutOfBoundsException`
  [Args.java:get].
- `getArgsClass` keys its lookup on the descriptor with the return type changed
  to `V`. Descriptors that differ only in return type therefore share one class
  [ArgsClassGenerator.java:getArgsClass].
- New class names have the form `SYNTHETIC_PACKAGE + ".Args$" + nextIndex`.
  `nextIndex` starts at 1 and is an instance field, so each generator numbers
  its classes from `Args$1` independently
  [ArgsClassGenerator.java:getArgsClass; ArgsClassGenerator.java:nextIndex].
- A repeat request for a known descriptor returns the existing info, so only
  the first requesting mixin is recorded [ArgsClassGenerator.java:getArgsClass].
- `generate` returns `false` and leaves `classNode` unchanged when `name` is not
  a dotted name that `getArgsClass` assigned [ArgsClassGenerator.java:generate].
- The generated class targets `V1_6` and has the access flags
  `ACC_PUBLIC | ACC_SUPER | ACC_SYNTHETIC`. Its superclass is `ARGS_REF` and its
  source file is the simple name plus `.java`. When
  `MixinEnvironment.Option.DEBUG_VERIFY` is on, output goes through a
  `CheckClassAdapter` [ArgsClassGenerator.java:generate].
- The generated constructor is private with the descriptor
  `([Ljava/lang/Object;)V`. The generated `toString()` returns `"Args"` plus the
  target signature [ArgsClassGenerator.java:generateCtor; generateToString].
- The generated `public static of(...)` takes the target's argument types,
  boxes each primitive through `<Box>.valueOf`, and returns the subclass type
  [ArgsClassGenerator.java:generateFactory].
- One getter is generated per argument. It is named `"$" + index`, has the
  descriptor `"()" + argDesc`, and unboxes primitives or casts references with
  `CHECKCAST` [ArgsClassGenerator.java:generateGetters].
- The generated `set(int, Object)`:
  - throws `ArgumentIndexOutOfBoundsException(index)` for any index outside
    `0..n-1`, including negative indices;
  - throws `NullPointerException("Argument with primitive type cannot be set to NULL")`
    when given `null` for a primitive slot;
  - throws a JVM `ClassCastException` when the value has the wrong type;
  - accepts `null` for a reference slot
  [ArgsClassGenerator.java:generateIndexedSetter].
- The generated `setAll(Object[])` throws
  `ArgumentCountException(values.length, expected, signature)` before it stores
  anything when the length is wrong. It stores values in index order, so a
  `null` for a primitive at index k throws `NullPointerException` after indices
  0 to k-1 are already written [ArgsClassGenerator.java:generateMultiSetter].
- Argument counts and indices are emitted as `byte` constants
  [ArgsClassGenerator.java:generateFactory].

## Side Effects

- Class initialisation gets a static logger from
  `MixinService.getService().getLogger("mixin")` [ArgsClassGenerator.java:logger].
- The first request for a new descriptor:
  - logs at DEBUG `"ArgsClassGenerator assigning {} for descriptor {}"`;
  - writes both internal maps and increments `nextIndex`;
  - calls `registry.accept(info)` exactly once
  [ArgsClassGenerator.java:getArgsClass].
- Each successful `generate` increments `ArgsClassInfo.loaded`. Generating a
  class again logs at DEBUG
  `"ArgsClassGenerator is re-generating {}, already did this {} times!"` and
  then regenerates it [ArgsClassGenerator.java:generate].
- The generator writes bytecode only into the `ClassNode` the caller supplies.
  It never loads or defines classes itself [ArgsClassGenerator.java:generate].

## Verification

- Compile: `mise exec -- ./gradlew compileJava` (run from the repository root).
- Test: `mise exec -- ./gradlew test`. `build.gradle` declares the `test`
  source set, but `src/test` does not exist yet, so there are no tests today.
