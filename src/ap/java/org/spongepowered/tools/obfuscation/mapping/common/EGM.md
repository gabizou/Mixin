# mapping/common

## Overview

Abstract base classes for the concrete mapping readers and writers. The SRG and
TSRG implementations in the sibling `mcp` and `fg3` packages extend them, and so
does the example JSON writer in the `example` source set. `MappingProvider` holds
four lookup tables. `MappingWriter` turns an output name into an open writer.

## Interface

- `MappingProvider`: public abstract class that implements `IMappingProvider`.
  The constructor is `(Messager, Filer)`. It has four protected
  `BiMap` fields: `packageMap`, `classMap`, `fieldMap` and `methodMap`.
  Subclasses implement `read(File)` [MappingProvider.java:MappingProvider].
- `MappingProvider.clear()`, `isEmpty()`, `getMethodMapping(MappingMethod)`,
  `getFieldMapping(MappingField)`, `getClassMapping(String)`,
  `getPackageMapping(String)` [MappingProvider.java:getMethodMapping].
- `MappingWriter`: public abstract class that implements `IMappingWriter`.
  The constructor is `(Messager, Filer)`. Subclasses implement
  `write(String, ObfuscationType, MappingSet, MappingSet)` [MappingWriter.java:MappingWriter].
- `MappingWriter.openFileWriter(String fileName, String description)`:
  protected, returns a `PrintWriter`, and throws `IOException` [MappingWriter.java:openFileWriter].
- `MappingWriter.printMessage(MessageType, CharSequence)`: protected [MappingWriter.java:printMessage].

## Invariants

- Each lookup is a plain map `get`. It returns `null` when the key is absent.
  This class does not search superclasses [MappingProvider.java:getMethodMapping].
- `isEmpty()` is true only when all four maps are empty [MappingProvider.java:isEmpty].
- `clear()` empties all four maps [MappingProvider.java:clear].
- Output path rule: a `fileName` that matches `^.*[\\/:].*$` (contains `/`,
  `\` or `:`) is treated as a filesystem path and opened with
  `new PrintWriter(File)`. Any other name is created with
  `Filer.createResource(StandardLocation.CLASS_OUTPUT, "", fileName)`
  [MappingWriter.java:openFileWriter].
- `printMessage` delegates to `IMessagerEx.printMessage(MessageType, ...)` when
  the messager is an `IMessagerEx`. Otherwise it prints only if
  `type.isEnabled()`, using `type.getKind()` and `type.decorate(msg)`
  [MappingWriter.java:printMessage].

## Side Effects

- For a filesystem path, `openFileWriter` creates the parent directories with
  `File.mkdirs()` and opens the file for writing [MappingWriter.java:openFileWriter].
- Both branches emit `MessageType.INFO` with the text
  `Writing <description> to <absolute path>` [MappingWriter.java:openFileWriter].
- `IOException` from `Filer.createResource` or `PrintWriter` reaches the caller
  [MappingWriter.java:openFileWriter].

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileApJava`
- Run: `mise exec -- ./gradlew test`. Note that `src/test` does not exist yet,
  so no tests cover this package today.
