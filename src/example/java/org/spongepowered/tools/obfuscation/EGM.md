# example: org/spongepowered/tools/obfuscation

## Overview

The package root for the sample obfuscation service in the `example` source
set. It has no source files of its own. Its package matches the annotation
processor base package in `src/ap/java/org/spongepowered/tools/obfuscation`,
so the sample sits next to the built-in `mcp` and `fg3` services.

## Subdirectories & Boundaries

- json: [./json/EGM.md](./json/EGM.md). The `ObfuscationServiceJson` sample
  service registers an obfuscation type named `json`. It reads `reobfSrgFile`
  SRG input and writes generated field and method mappings as pretty-printed
  JSON to the path in the `outputJson` option.
