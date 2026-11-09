# org.spongepowered.asm.util.throwables

## Overview

Package `org.spongepowered.asm.util.throwables` holds the exception and error types thrown by helpers in `org.spongepowered.asm.util`. They cover constraint parsing and checking (`ConstraintParser`), local variable table generation (`Locals`), and conflicts between synthetic bridge methods during mixin merging.

## Interface

- `ConstraintViolationException extends Exception` is checked [ConstraintViolationException.java:class].
  - It has eight constructors: `(Constraint)`, `(Constraint, int badValue)`, `(String, Constraint)`, `(String, Constraint, int)`, `(Throwable, Constraint)`, `(Throwable, Constraint, int)`, `(String, Throwable, Constraint)` and `(String, Throwable, Constraint, int)` [ConstraintViolationException.java:ConstraintViolationException].
  - The forms that take only a constraint call `Exception()`. The `(Throwable, ...)` forms call `Exception(Throwable)`, so `getMessage()` returns `cause.toString()` [ConstraintViolationException.java:ConstraintViolationException].
  - `getConstraint()` returns the constraint passed in, which may be `null` [ConstraintViolationException.java:getConstraint].
  - `getBadValue()` returns `String.valueOf(badValue)` when an int was supplied, and `"UNRESOLVED"` otherwise [ConstraintViolationException.java:getBadValue].
- `InvalidConstraintException extends IllegalArgumentException` is unchecked. Its constructors `()`, `(String)`, `(Throwable)` and `(String, Throwable)` delegate unchanged [InvalidConstraintException.java:class].
- `LVTGeneratorError extends org.spongepowered.asm.mixin.throwables.MixinError`, which is a `java.lang.Error`. Its constructors are `(String)` and `(String, Throwable)` [LVTGeneratorError.java:class].
- `SyntheticBridgeException extends org.spongepowered.asm.mixin.throwables.MixinException`, which is unchecked [SyntheticBridgeException.java:class].
  - `enum Problem`, in declaration order: `BAD_INSN`, `BAD_LOAD`, `BAD_CAST`, `BAD_INVOKE_NAME`, `BAD_INVOKE_DESC`, `BAD_LENGTH`. Each value holds a positional `String.format` template [SyntheticBridgeException.java:Problem].
  - `SyntheticBridgeException(Problem, String name, String desc, int index, AbstractInsnNode a, AbstractInsnNode b)` builds the message from `problem.getMessage(...)`. A `null` problem throws `NullPointerException`. There are no public getters for the stored fields [SyntheticBridgeException.java:SyntheticBridgeException].
  - Template arguments `%1$` to `%9$` are, in order: `name`, `desc`, `index`, `opcodeName(a)`, `opcodeName(a)`, `insnName(a)`, `insnName(b)`, `insnDesc(a)`, `insnDesc(b)`. Argument `%5$s` is the opcode of `a`, not `b`, so a `BAD_INSN` message repeats `a`'s opcode. `insnName` and `insnDesc` read a `MethodInsnNode` and return `""` for any other node. `Bytecode.getOpcodeName(null)` returns `""` [SyntheticBridgeException.java:Problem.getMessage].
  - Example `BAD_LENGTH` template: `"Mismatched bridge method length for synthetic bridge method %1$s%2$s unexpected extra opcode at offset %3$d"` [SyntheticBridgeException.java:Problem].
  - `printAnalysis(IMixinContext, MethodNode mda, MethodNode mdb)` builds an `org.spongepowered.asm.util.PrettyPrinter` report and prints it to `System.err` [SyntheticBridgeException.java:printAnalysis]:
    - The report contains the message, the method, the problem type, and the "Existing method" listing (its owner is taken from `@MixinMerged(mixin)` or else the context target class).
    - It then adds the "Incoming method" listing, with the instruction at `index` marked `>>>>`, and the per-problem "Analysis".
  - Analysis failure cases:
    - `BAD_CAST` and `BAD_INVOKE_DESC` cast `a` and `b` (to `TypeInsnNode` and `MethodInsnNode` respectively) and throw `ClassCastException` for other node types.
    - `BAD_LOAD` indexes `argTypes[var-1]` without adjusting for wide slots, so it can throw `ArrayIndexOutOfBoundsException` [SyntheticBridgeException.java:printProblem].

## Invariants

- `ConstraintViolationException.getBadValue()` never returns `null` [ConstraintViolationException.java:badValue].
- `SyntheticBridgeException`'s message is fixed at construction. `MixinException` adds an activity suffix only if `prepend(...)` is called later [SyntheticBridgeException.java:SyntheticBridgeException].
- `InvalidConstraintException` and `LVTGeneratorError` add no state or behaviour to their superclasses [InvalidConstraintException.java:class] [LVTGeneratorError.java:class].

## Side Effects

- `SyntheticBridgeException.printAnalysis` writes a multi-line report to `System.err` [SyntheticBridgeException.java:printAnalysis].
- For `BAD_LOAD`, `BAD_CAST` and `BAD_INVOKE_DESC`, the analysis calls `ClassInfo.getCommonSuperClassOrInterface`, which uses the global `org.spongepowered.asm.mixin.transformer.ClassInfo` cache [SyntheticBridgeException.java:printProblem].
- The constructors and `getMessage` have no side effects. `ConstraintViolationException`, `InvalidConstraintException` and `LVTGeneratorError` have none at all.

## Verification

Run these from the repository root:

- `mise exec -- ./gradlew compileJava`
- `mise exec -- ./gradlew test`

`build.gradle` declares a `test` source set, but `src/test` does not exist yet, so there are no tests today. Good unit-test targets are the formatting of `SyntheticBridgeException` messages for each `Problem`, including the repeated `a` opcode in `BAD_INSN`, and `ConstraintViolationException.getBadValue()` with and without an int.
