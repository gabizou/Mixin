# org.spongepowered.asm.transformers

## Overview

This package holds ASM helper base classes used by the mixin processor:

- `MixinClassReader` fails fast, with a clear message, on class files newer than the active ASM supports.
- `MixinClassWriter` finds common superclasses from Mixin's `ClassInfo` metadata instead of loading classes.
- `TreeTransformer` is an `ILegacyClassTransformer` base with read and write helpers for the tree API.

## Interface

- `public class MixinClassReader extends ClassReader`, with constructor `MixinClassReader(byte[] classFile, String name)` [MixinClassReader.java:MixinClassReader]
- `public class MixinClassWriter extends ClassWriter` [MixinClassWriter.java:MixinClassWriter]:
  - constructors `MixinClassWriter(int flags)` and `MixinClassWriter(ClassReader classReader, int flags)`
  - `protected String getCommonSuperClass(String type1, String type2)`, which is overridden [MixinClassWriter.java:getCommonSuperClass]
- `public abstract class TreeTransformer implements ILegacyClassTransformer` [TreeTransformer.java:TreeTransformer]:
  - `protected final ClassNode readClass(String className, byte[] basicClass)`
  - `protected final ClassNode readClass(String className, byte[] basicClass, boolean cacheReader)`
  - `protected final byte[] writeClass(ClassNode classNode)`
  - `ILegacyClassTransformer` is in src/main/java/org/spongepowered/asm/service.

## Invariants

### MixinClassReader

- Before calling the `ClassReader` constructor, it reads the major version from bytes 6 and 7 of the class file. If that version is greater than `ASM.getMaxSupportedClassVersionMajor()`, it throws `IllegalArgumentException` with this message, where `%d` is a number and `%s` is the class name [MixinClassReader.java:checkClassVersion]:

  ```text
  Class file major version %d is not supported by active ASM (version %d.%d supports class version %d), reading %s
  ```

- An array shorter than 8 bytes throws `ArrayIndexOutOfBoundsException` before that check [MixinClassReader.java:checkClassVersion].

### MixinClassWriter

- `getCommonSuperClass(a, b)` returns `ClassInfo.getCommonSuperClass(a, b).getName()` and never calls `Class.forName` [MixinClassWriter.java:getCommonSuperClass].

### TreeTransformer

- `readClass` always creates a `MixinClassReader`, so it applies the same version check. It accepts into a new `ClassNode` with `ClassReader.EXPAND_FRAMES` [TreeTransformer.java:readClass].
- With `cacheReader` true, which is the default for the two-argument overload, the reader is stored in the `classReader` field [TreeTransformer.java:readClass].
- `writeClass` writes with `COMPUTE_MAXS | COMPUTE_FRAMES`. It takes the cached-reader branch only when `classReader != null && this.classNode == classNode` [TreeTransformer.java:writeClass].
- No code assigns the `classNode` field a non-null value; it is only ever set to null. So for any non-null node, `writeClass` uses the plain `new MixinClassWriter(flags)` branch, and the cached `classReader` is not cleared on that branch [TreeTransformer.java:classNode] [TreeTransformer.java:writeClass].

## Side Effects

- `TreeTransformer` keeps per-instance mutable state (`classReader` and `classNode`) with no synchronization, so one instance is not safe to use from several threads at once [TreeTransformer.java:classReader].
- `MixinClassWriter.getCommonSuperClass` triggers `ClassInfo` metadata lookups (src/main/java/org/spongepowered/asm/mixin/transformer/ClassInfo.java), which may load class bytecode through the active service [MixinClassWriter.java:getCommonSuperClass].
- The package has no static state.

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileJava`
- Run from the repo root: `mise exec -- ./gradlew test`
- `build.gradle` declares a `test` source set, but `src/test` does not exist yet. `test` therefore runs no tests today.
