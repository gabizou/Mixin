# struct

## Overview

Small value types for diagnostics. `Message` holds one diagnostic so it can be
queued and sent later. `InjectorRemap` holds an injector's remap flag and a
pending "no mapping" error. The error is dropped if any `@At` in the same
injector is remapped.

## Interface

- `Message`: public class.
  - Twelve constructors combine a `MessageType` or `Diagnostic.Kind`, a
    `CharSequence` message, and optionally an `Element`, an
    `AnnotationHandle` or `AnnotationMirror`, and an `AnnotationValue`.
  - Methods: `sendTo(Messager)`, `getKind()`, `setKind(Kind)`, `getMsg()`,
    `setMsg(CharSequence)`, `getElement()`, `getAnnotation()` and `getValue()`.
  [Message.java:Message]
- `InjectorRemap`: public class.
  - Constructor: `(boolean remap)`.
  - Methods: `shouldRemap()`, `notifyRemapped()`,
    `addMessage(MessageType, CharSequence, Element, AnnotationHandle)`,
    `clearMessage()` and `dispatchPendingMessages(Messager)`.
  [InjectorRemap.java:InjectorRemap]

## Invariants

- A `Message` built from a `MessageType` stores `type.getKind()` and
  `type.decorate(msg)` at construction. It never checks `type.isEnabled()`
  [Message.java:Message].
- `sendTo` always prints. It calls the most specific `Messager.printMessage`
  overload that its fields allow: value, then annotation, then element, then
  message only [Message.java:sendTo].
- `InjectorRemap.addMessage` replaces any pending message, so only the last
  one is kept [InjectorRemap.java:addMessage].
- `notifyRemapped()` increments the remapped count and clears the pending
  message [InjectorRemap.java:notifyRemapped].
- `dispatchPendingMessages` sends the pending message only when no `@At` was
  remapped and a message is pending [InjectorRemap.java:dispatchPendingMessages].
- Because of the first point, an injector's `NO_OBFDATA_FOR_TARGET` or
  `NO_OBFDATA_FOR_CTOR` error is sent through `InjectorRemap` even when that
  `MessageType` is disabled with `MSG_<NAME>=disabled`. The kind is still
  applied. Sources: `src/ap/java/org/spongepowered/tools/obfuscation/AnnotatedMixinElementHandlerInjector.java`
  and `src/ap/java/org/spongepowered/tools/obfuscation/AnnotatedMixins.java` (`registerInjector`).

## Side Effects

- `Message.sendTo` writes a compiler diagnostic through the supplied
  `Messager` [Message.java:sendTo].
- `Message.setKind` and `setMsg` change the message in place [Message.java:setKind].

## Verification

- Run from the repo root: `mise exec -- ./gradlew compileApJava`
- Run: `mise exec -- ./gradlew test`. Note that `src/test` does not exist yet,
  so no tests cover this package today.
