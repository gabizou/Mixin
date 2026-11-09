# org.spongepowered.asm.lib (legacy source set)

## Overview

This package holds binary-compatibility shims. They recreate the Mixin 0.7.x
repackaged ASM type names that the `ClassNode` shim in `tree` needs in order to
resolve. Each class's Javadoc is "Shim, see {@link ClassNode}". None of them
wraps or delegates to real ASM. `ClassVisitor` throws `CompanionPluginError`
from every method, and the other five types are empty.

## Interface

- [tree](./tree/EGM.md): the `ClassNode` compatibility shim, which extends
  `ClassVisitor` from this package.
- `public abstract class ClassVisitor` has no fields and no declared
  constructor [ClassVisitor.java:ClassVisitor]. Each method below is public and
  throws `new CompanionPluginError("ClassVisitor.<name>")`:

| Method | Signature | Thrown message |
| --- | --- | --- |
| `visit` | `void visit(int version, int access, String name, String signature, String superName, String[] interfaces)` | `ClassVisitor.visit` |
| `visitSource` | `void visitSource(String source, String debug)` | `ClassVisitor.visitSource` |
| `visitOuterClass` | `void visitOuterClass(String owner, String name, String desc)` | `ClassVisitor.visitOuterClass` |
| `visitAnnotation` | `AnnotationVisitor visitAnnotation(String desc, boolean visible)` | `ClassVisitor.visitAnnotation` |
| `visitTypeAnnotation` | `AnnotationVisitor visitTypeAnnotation(int typeRef, TypePath typePath, String desc, boolean visible)` | `ClassVisitor.visitTypeAnnotation` |
| `visitAttribute` | `void visitAttribute(Attribute attr)` | `ClassVisitor.visitAttribute` |
| `visitInnerClass` | `void visitInnerClass(String name, String outerName, String innerName, int access)` | `ClassVisitor.visitInnerClass` |
| `visitField` | `FieldVisitor visitField(int access, String name, String desc, String signature, Object value)` | `ClassVisitor.visitField` |
| `visitMethod` | `MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions)` | `ClassVisitor.visitMethod` |
| `visitEnd` | `void visitEnd()` | `ClassVisitor.visitEnd` |

- Five empty placeholder types exist only so that the `ClassVisitor` method
  signatures resolve:
  - `public abstract class AnnotationVisitor` [AnnotationVisitor.java:AnnotationVisitor]
  - `public class Attribute` [Attribute.java:Attribute]
  - `public abstract class FieldVisitor` [FieldVisitor.java:FieldVisitor]
  - `public abstract class MethodVisitor` [MethodVisitor.java:MethodVisitor]
  - `public class TypePath` [TypePath.java:TypePath]

## Invariants

- Every `ClassVisitor` method throws unconditionally. None of them returns a
  value or does nothing [ClassVisitor.java:visit] [ClassVisitor.java:visitEnd].
- `AnnotationVisitor`, `Attribute`, `FieldVisitor`, `MethodVisitor` and
  `TypePath` declare no members. `Attribute` and `TypePath` are concrete with
  only the implicit no-arg constructor. The other three are abstract.
- These files import only `org.spongepowered.asm.lib.tree.ClassNode`, which is
  used for the Javadoc link. `ClassVisitor` also imports
  `org.spongepowered.asm.mixin.throwables.CompanionPluginError`. Nothing here
  references `org.objectweb.asm` [ClassVisitor.java:import].
- These types exist only for binary compatibility. src/main contains no
  subclass of them and no call to them. The only use of the legacy packages in
  src/main is src/main/java/org/spongepowered/asm/mixin/transformer/PluginHandle.java,
  which uses `org.spongepowered.asm.lib.tree.ClassNode` and
  `CompanionPluginError`.
- Every file in the package starts with a `CHECKSTYLE:OFF` comment.

## Side Effects

- Each `ClassVisitor` method throws `CompanionPluginError`, which is a
  `java.lang.LinkageError`, with the message shown in the table
  [ClassVisitor.java:visit] [ClassVisitor.java:visitEnd]. There are no other
  side effects.

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileLegacyJava`.
- The `legacy` source set is declared at build.gradle:85-88, compiled at
  `compatibility = '1.6'` on a Java 8 toolchain (`languageVersion = 8`). Its only
  dependency is `org.ow2.asm:asm-tree:$asmVersion` (build.gradle:214).
- No tests cover this package at this commit.
