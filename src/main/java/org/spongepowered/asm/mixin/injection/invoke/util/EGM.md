# invoke/util

## Overview

Bytecode analysis helpers for invoke injectors. `InsnFinder` runs an ASM
`Analyzer` with a `BasicInterpreter` over a target method to find the
instruction that pops the value a given instruction pushes. Its caller is
`ModifyConstantInjector.checkNarrowing` in
src/main/java/org/spongepowered/asm/mixin/injection/invoke/ModifyConstantInjector.java.

## Interface

- `public class InsnFinder` has an implicit public no-argument constructor
  [InsnFinder.java:InsnFinder].
- `public AbstractInsnNode findPopInsn(Target target, AbstractInsnNode node)`
  returns the instruction that pops `node`'s pushed value, or `null`
  [InsnFinder.java:findPopInsn].
- Package-private helpers:
  - `AnalysisResultException extends RuntimeException` carries the result
    through `getResult()` [InsnFinder.java:AnalysisResultException].
  - `enum AnalyzerState` has the constants `SEARCH`, `ANALYSE` and `COMPLETE`
    [InsnFinder.java:AnalyzerState].
  - `PopAnalyzer extends Analyzer<BasicValue>` creates `PopFrame` frames
    [InsnFinder.java:PopAnalyzer; InsnFinder.java:PopAnalyzer.PopFrame].

## Invariants

- A `PopFrame` starts in `SEARCH` with depth 0. The first push while the
  current instruction is `node` switches it to `ANALYSE` with depth 1, and each
  later push adds 1 [InsnFinder.java:PopFrame.push].
- Pops in `ANALYSE` lower the depth. When depth reaches 0 the frame moves to
  `COMPLETE` and throws `AnalysisResultException(current)`. Pops in `SEARCH` or
  `COMPLETE` are not counted [InsnFinder.java:PopFrame.pop].
- The analysis runs as `analyze(target.classNode.name, target.method)`. A result
  is returned only from an `AnalyzerException` whose cause is an
  `AnalysisResultException` [InsnFinder.java:findPopInsn].
- `findPopInsn` returns `null` when the analysis finishes without a matching
  pop or fails with any other `AnalyzerException` [InsnFinder.java:findPopInsn].

## Side Effects

- Class initialisation gets a static logger from
  `MixinService.getService().getLogger("mixin")` [InsnFinder.java:logger].
- An `AnalyzerException` without an `AnalysisResultException` cause is logged
  through `logger.catching(ex)` and then swallowed [InsnFinder.java:findPopInsn].

## Verification

- Compile: `mise exec -- ./gradlew compileJava` (run from the repository root).
- Test: `mise exec -- ./gradlew test`. `build.gradle` declares the `test`
  source set, but `src/test` does not exist yet, so there are no tests today.
