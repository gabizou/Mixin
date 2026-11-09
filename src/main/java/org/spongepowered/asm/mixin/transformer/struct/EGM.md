# transformer/struct

## Overview

This package holds two value and helper types for constructor initialisers.

- `InsnRange` describes a line-number range plus an instruction-index marker. It can filter an ASM `InsnList` by that range.
- `Initialiser` takes the field-initialiser instructions out of a mixin constructor and validates them. It then clones them into the target constructors.

## Interface

- `InsnRange` has public final fields `start`, `end` (line numbers) and `marker` (instruction index). Constructor: `InsnRange(int start, int end, int marker)` [InsnRange.java:InsnRange]
- `InsnRange.isValid()` returns `start != 0 && end != 0 && end >= start` [InsnRange.java:isValid]
- `InsnRange.contains(int)` and `excludes(int)` are inclusive at both ends [InsnRange.java:contains] [InsnRange.java:excludes]
- `InsnRange.apply(InsnList, boolean inclusive)` returns a new `ArrayDeque` of the original node references, not clones [InsnRange.java:apply]
- `Initialiser(MixinTargetContext, MethodNode ctor, InsnRange)`. Methods: `size()`, `getMaxStack()`, `getCtor()`, `getInsns()` (returns the live deque), `injectInto(org.spongepowered.asm.mixin.injection.struct.Constructor)` [Initialiser.java:Initialiser]
- `Initialiser.InjectionMode` enum: `DEFAULT`, `SAFE`. `static ofEnvironment(MixinEnvironment)` [Initialiser.java:InjectionMode]
- `Initialiser.OPCODE_BLACKLIST` (protected static) lists 26 opcodes: `RETURN`, the primitive `*LOAD` opcodes, all `*ALOAD`, the `*STORE` opcodes including `ASTORE`, and all `*ASTORE`. `ALOAD` is not in the list [Initialiser.java:OPCODE_BLACKLIST]

## Invariants

- `apply` behaviour [InsnRange.java:apply]:
  - Iteration starts at index `marker`.
  - Nothing is gathered before the first `LineNumberNode`.
  - `LineNumberNode`s themselves are never added.
  - A `LineNumberNode` that is the last instruction throws `IndexOutOfBoundsException`, from the `get(index + 1)` lookahead.
- At the `end` line, `apply` checks whether the next instruction is `RETURN`:
  - If it is not, gathering is set to `!inclusive` and the first following `RETURN` is trimmed.
  - Otherwise gathering uses `contains(line)` when `inclusive` and `excludes(line)` when not.
  - Outside trim mode, gathered nodes with opcode `-1` that are not labels, such as `FrameNode`, are dropped.

  [InsnRange.java:apply]
- `apply` holds back a `LabelNode` and adds it only when a later instruction is gathered [InsnRange.java:apply]
- `InsnRange` does not override `equals` or `hashCode`, so instances compare by identity [InsnRange.java:InsnRange]
- The `Initialiser` constructor extracts `range.apply(ctor.instructions, false)` and then validates it:
  - An opcode in `OPCODE_BLACKLIST` throws `InvalidMixinException("Cannot handle <OPCODE> opcode (0x..) in class initialiser")`.
  - A non-empty result that does not end in `PUTFIELD` throws `InvalidMixinException("Could not parse initialiser, expected 0xB5, found 0x..")`.
  - An empty result is accepted.

  [Initialiser.java:initInstructions]
- `InjectionMode.ofEnvironment` reads `Option.INITIALISER_INJECTION_MODE` case-insensitively. A `null` value gives `DEFAULT`. An unparseable value logs a warning and also gives `DEFAULT` [Initialiser.java:InjectionMode.ofEnvironment]
- `injectInto` behaviour [Initialiser.java:injectInto]:
  - If `ctor.findInitialiserInjectionPoint(mode)` returns `null`, it logs a warning and injects nothing.
  - `LabelNode`s are skipped.
  - A `JumpInsnNode` throws `InvalidMixinException("Unsupported JUMP opcode in initialiser in ...")`.
  - Every other node is cloned with the target constructor's label map.

## Side Effects

- Loading the `Initialiser` class calls `MixinService.getService().getLogger("mixin")` [Initialiser.java:logger]
- `injectInto` mutates the target `Constructor` through `insertBefore`. It is not atomic: nodes inserted before a rejected jump stay in place [Initialiser.java:injectInto]
- `InsnRange.apply` never modifies the input list [InsnRange.java:apply]

## Verification

Run from the repository root:

- `mise exec -- ./gradlew compileJava`
- `mise exec -- ./gradlew test`

The `test` source set is declared in `build.gradle`, but `src/test` does not exist yet, so there are no tests today.
