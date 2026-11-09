# org.spongepowered.asm.lib.tree (legacy source set)

## Overview

This package holds one binary-compatibility shim, `ClassNode`. It keeps 0.7.x
`IMixinConfigPlugin` companions linking under 0.8. In Mixin 0.7.x, ASM was
repackaged as `org.spongepowered.asm.lib`. The class Javadoc says companions
compiled for 0.7.x "will still work with 0.8 but will only crash if they access
members of the ClassNode provided in preApply and postApply"
[ClassNode.java:ClassNode]. The shim is a read-only snapshot of eleven header
fields of a native `org.objectweb.asm.tree.ClassNode`, and both of its own
methods throw `CompanionPluginError`.

The only caller is outside this partition, in
src/main/java/org/spongepowered/asm/mixin/transformer/PluginHandle.java. When a
plugin's `preApply` or `postApply` throws `AbstractMethodError`, `PluginHandle`
switches to `CompatibilityMode.COMPATIBLE`. It then finds the plugin's methods
by reflection using this `ClassNode` type (`PluginHandle.initReflection`), and
invokes them with `new org.spongepowered.asm.lib.tree.ClassNode(targetClass)`
(`PluginHandle.applyLegacy`).

## Interface

- `public class ClassNode extends org.spongepowered.asm.lib.ClassVisitor`
  [ClassNode.java:ClassNode]. The class is concrete. Its superclass is the shim
  in src/legacy/java/org/spongepowered/asm/lib/ClassVisitor.java.
- `public ClassNode(org.objectweb.asm.tree.ClassNode classNode)` is the only
  constructor. It assigns the eleven fields below and does nothing else
  [ClassNode.java:ClassNode(org.objectweb.asm.tree.ClassNode)].
- The class has eleven public final fields. Each is set from the field of the
  same name on the wrapped native node:

| Field | Type | Value |
| --- | --- | --- |
| `version` | `int` | copied |
| `access` | `int` | copied |
| `name` | `String` | same reference |
| `signature` | `String` | same reference |
| `superName` | `String` | same reference |
| `interfaces` | `List<String>` | `Collections.unmodifiableList(classNode.interfaces)`, a view and not a copy |
| `sourceFile` | `String` | same reference |
| `sourceDebug` | `String` | same reference |
| `outerClass` | `String` | same reference |
| `outerMethod` | `String` | same reference |
| `outerMethodDesc` | `String` | same reference |

- `public void check(final int api)` always throws
  `new CompanionPluginError("ClassNode.check")` [ClassNode.java:check].
- `public void accept(final ClassVisitor cv)` always throws
  `new CompanionPluginError("ClassNode.accept")`. The parameter type is the shim
  `org.spongepowered.asm.lib.ClassVisitor` [ClassNode.java:accept].
- No `visit*` method is overridden. Each inherited one throws
  `CompanionPluginError("ClassVisitor.<method>")`
  (src/legacy/java/org/spongepowered/asm/lib/ClassVisitor.java).

## Invariants

- All eleven fields are `final`, so code that writes to one does not compile
  [ClassNode.java:version]. The Javadoc puts this as "any attempt to *write* to
  a field will fail".
- Reading the eleven fields never throws. Only the methods throw
  [ClassNode.java:check] [ClassNode.java:accept]. The Javadoc also says "any
  access to fields or methods will raise an error", but in the code this is true
  only of the methods.
- The shim copies no member lists, such as methods, fields, annotations or inner
  classes. Its only fields are the eleven listed above [ClassNode.java:ClassNode].
- `interfaces` is a live read-only view of the wrapped node's list
  [ClassNode.java:interfaces]:
  - Mutating calls throw `UnsupportedOperationException`.
  - Later changes to `classNode.interfaces` show through.
  - A `null` source list makes the constructor throw `NullPointerException`.
- The file uses only `java.util`, the fully qualified
  `org.objectweb.asm.tree.ClassNode`, and the legacy packages
  [ClassNode.java:import].

## Side Effects

- None, apart from the throws listed under Interface. The constructor only
  assigns fields, and it keeps a reference to the wrapped node's `interfaces`
  list [ClassNode.java:ClassNode(org.objectweb.asm.tree.ClassNode)].

## Verification

- Run from the repository root: `mise exec -- ./gradlew compileLegacyJava`.
- The `legacy` source set is declared at build.gradle:85-88 with
  `languageVersion = 8` and `compatibility = '1.6'`. build.gradle:320-323
  applies the compatibility value as the source and target compatibility.
- Its only dependency is `legacyImplementation
  "org.ow2.asm:asm-tree:$asmVersion"` (build.gradle:214), with `asmVersion=9.5`
  (gradle.properties:9).
- No tests cover this package at this commit.
