# org.spongepowered.asm.mixin.struct

## Overview

This package holds internal data bundles the mixin processor uses while it applies mixins:

- `AnnotatedMethodInfo` and `SpecialMethodInfo` describe an annotated mixin method. `SpecialMethodInfo` is the base of `AccessorInfo` in the sibling package `mixin/gen` and of the injector infos in src/main/java/org/spongepowered/asm/mixin/injection.
- `SourceMap` builds JSR-45 `SMAP` text for the `SourceDebugExtension` attribute.
- `MemberRef` gives one view over a field instruction, a method instruction or a method handle.

## Interface

- `public class AnnotatedMethodInfo implements IInjectionPointContext` [AnnotatedMethodInfo.java:AnnotatedMethodInfo]. `IInjectionPointContext` is in src/main/java/org/spongepowered/asm/mixin/injection.
  - `public AnnotatedMethodInfo(IMixinContext mixin, MethodNode method, AnnotationNode annotation)`
  - `final String getElementDescription()`, `String remap(String reference)` and `ISelectorContext getParent()` (always null)
  - `final IMixinContext getMixin()`, `final MethodNode getMethod()` and `String getMethodName()`
  - `AnnotationNode getAnnotationNode()`, `final IAnnotationHandle getAnnotation()` and `IAnnotationHandle getSelectorAnnotation()`
  - `String getSelectorCoordinate(boolean leaf)` and `void addMessage(String format, Object... args)`
  - `public static final String getDynamicInfo(Object | MethodNode | IAnnotatedElement method)` [AnnotatedMethodInfo.java:getDynamicInfo]
- `public class SpecialMethodInfo extends AnnotatedMethodInfo` [SpecialMethodInfo.java:SpecialMethodInfo]:
  - `public SpecialMethodInfo(MixinTargetContext mixin, MethodNode method, AnnotationNode annotation)`
  - `@Deprecated final ClassNode getClassNode()` and `final ClassNode getTargetClassNode()`
  - `final ClassInfo getTargetClassInfo()`, `final ClassInfo getClassInfo()` and `String getMethodName()`
- `public class SourceMap` [SourceMap.java:SourceMap]:
  - `public SourceMap(String sourceFile)`, `String getSourceFile()` and `String getPseudoGeneratedSourceFile()`
  - `File addFile(ClassNode)`, `File addFile(String stratumName, ClassNode)`, `File addFile(String sourceFileName, String sourceFilePath, int size)` and `File addFile(String stratumName, String sourceFileName, String sourceFilePath, int size)`
  - `String toString()`, which returns the SMAP text
  - Nested `public static class SourceMap.File` has the public final fields `id`, `lineOffset`, `size`, `sourceFileName` and `sourceFilePath`, plus `applyOffset(ClassNode)`, `applyOffset(MethodNode)` and `appendLines(StringBuilder)` [SourceMap.java:File]
  - Nested `static class Stratum` is package-private [SourceMap.java:Stratum]
- `public abstract class MemberRef` [MemberRef.java:MemberRef]:
  - Abstract accessors: `boolean isField()`, `int getOpcode()`/`void setOpcode(int)`, `String getOwner()`/`setOwner(String)`, `String getName()`/`setName(String)` and `String getDesc()`/`setDesc(String)`
  - `toString()`, `equals(Object)` and `hashCode()`
  - Nested `public static final class Method(MethodInsnNode insn)`, with a public final field `insn` [MemberRef.java:Method]
  - Nested `public static final class Field(FieldInsnNode insn)`, with a public final field `insn` [MemberRef.java:Field]
  - Nested `public static final class Handle(org.objectweb.asm.Handle handle)`, with `getMethodHandle()` and `setHandle(int tag, String owner, String name, String desc, boolean isInterface)` [MemberRef.java:Handle]

## Invariants

### AnnotatedMethodInfo and SpecialMethodInfo

- `annotationType` is `"@" + <annotation simple name>`, or `"Undecorated method"` when the annotation is null. `getElementDescription()` returns `"<annotationType> annotation on <methodName>"` [AnnotatedMethodInfo.java:getElementDescription].
- `methodName` is the original name, `MethodNodeEx.getName(method)`. `AnnotatedMethodInfo.getMethodName()` returns the current `method.name`, but `SpecialMethodInfo.getMethodName()` returns the original `methodName` [AnnotatedMethodInfo.java:getMethodName] [SpecialMethodInfo.java:getMethodName].
- `remap(reference)` returns `reference` unchanged when the context or its refmap is null. Otherwise it returns `refMap.remap(context.getClassRef(), reference)` [AnnotatedMethodInfo.java:remap].
- `getSelectorCoordinate(true)` returns `"method"`. `getSelectorCoordinate(false)` returns `getMethodName().toLowerCase(Locale.ROOT)` [AnnotatedMethodInfo.java:getSelectorCoordinate].
- `addMessage` prints only when `context.getOption(Option.DEBUG_VERBOSE)` is true. It does not check the context for null [AnnotatedMethodInfo.java:addMessage].
- `getDynamicInfo` returns `""` in two cases: no `@Dynamic` is present (for a `MethodNode`, only the invisible annotation is checked), or the argument type is unsupported. Otherwise it returns `" Method is @Dynamic(<desc>)."`, where `<desc>` is the annotation value. When a `mixin` type value is present, `<desc>` is prefixed with `{<class name>} `. An empty `<desc>` still gives `""` [AnnotatedMethodInfo.java:getDynamicInfo].
- `SpecialMethodInfo` captures `classNode = mixin.getTargetClassNode()` once, at construction [SpecialMethodInfo.java:SpecialMethodInfo].

### SourceMap

`toString()` emits exactly this text, with every line ending in `\n` [SourceMap.java:appendTo] [SourceMap.java:Stratum.appendTo] [SourceMap.java:File.appendFile] [SourceMap.java:File.appendLines]:

```text
SMAP
<sourceFile>
Mixin
*S <stratum name>
*F
<id> <sourceFileName>
*L
1#<id>,<size>:<lineOffset>
*E
```

- The `*S`, `*F` and `*L` blocks repeat for each stratum. A file with a non-null `sourceFilePath` is written as `+ <id> <sourceFileName>` followed by a line holding the path.
- The default stratum is always `"Mixin"`, and there is no setter for it [SourceMap.java:DEFAULT_STRATUM].
- `nextLineOffset` starts at 1. Every `addFile` call advances it by `size`, even when the stratum already has a `File` for the same `sourceFilePath`. In that case the existing `File` comes back unchanged [SourceMap.java:addFile] [SourceMap.java:Stratum.addFile].
- Within a stratum, files are keyed by `sourceFilePath`. Ids are 1-based and follow insertion order [SourceMap.java:Stratum.addFile].
- `addFile(stratum, ClassNode)` uses `classNode.sourceFile` as the file name, `classNode.name + ".java"` as the path, and `Bytecode.getMaxLineNumber(classNode, 500, 50)` as the size [SourceMap.java:addFile].
- `getPseudoGeneratedSourceFile()` returns `sourceFile.replace(".java", "$mixin.java")` [SourceMap.java:getPseudoGeneratedSourceFile].
- `File.applyOffset` adds `lineOffset - 1` to every `LineNumberNode.line` [SourceMap.java:File.applyOffset].

### MemberRef

- `equals` compares the opcode, owner, name and desc through the getters, so instances of different subclasses can be equal. A null owner, name or desc throws `NullPointerException` [MemberRef.java:equals].
- `hashCode()` is `toString().hashCode()`. `toString()` gives `"<OPCODE> for <owner>.<name><desc>"`, with a `:` before the desc for fields, for example `GETFIELD for a/B.x:I` [MemberRef.java:toString] [MemberRef.java:hashCode].
- `Method.setOpcode` and `Field.setOpcode` check `(opcode & OPCODES) == 0`, where `OPCODES` is the bitwise OR of the valid opcodes: `0xBF` for methods and `0xB7` for fields. Only an opcode that shares no bit with the mask throws `IllegalArgumentException("Invalid opcode for method|field instruction: 0x<hex>")`. For example, `Field.setOpcode(INVOKEVIRTUAL)` is accepted [MemberRef.java:Method.setOpcode] [MemberRef.java:Field.setOpcode].
- `Handle.getOpcode()` throws `MixinTransformerError("Invalid tag <tag> for method handle <handle>.")` when `Handles.opcodeFromTag` returns 0 [MemberRef.java:Handle.getOpcode].
- `Handle.setOpcode()` throws `MixinTransformerError("Invalid opcode <name> for method handle <handle>.")` when `Handles.tagFromOpcode` returns 0 [MemberRef.java:Handle.setOpcode].
- `Handle.isField()` delegates to `Handles.isField(handle)` [MemberRef.java:Handle.isField].

## Side Effects

- The `MemberRef.Method` and `MemberRef.Field` setters write straight to the wrapped `MethodInsnNode` or `FieldInsnNode` [MemberRef.java:Method] [MemberRef.java:Field].
- The `MemberRef.Handle` setters replace the private `handle` field with a new `org.objectweb.asm.Handle`. The original `Handle` object does not change, so callers must read `getMethodHandle()` back [MemberRef.java:Handle.setHandle].
- `SourceMap.File.applyOffset` changes the line numbers in the given `ClassNode` or `MethodNode` in place [SourceMap.java:File.applyOffset].
- `AnnotatedMethodInfo.addMessage` sends a `Kind.WARNING` through `MessageRouter.getMessager()` [AnnotatedMethodInfo.java:addMessage].
- The package has no mutable static state. Its static fields are only constants.

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileJava`
- Run from the repo root: `mise exec -- ./gradlew test`
- `build.gradle` declares a `test` source set, but `src/test` does not exist yet. `test` therefore runs no tests today.
