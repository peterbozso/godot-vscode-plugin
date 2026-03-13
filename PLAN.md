# Issue #924: Cannot open some documents

**URL:** https://github.com/godotengine/godot-vscode-plugin/issues/924
**Status:** Open
**Label:** bug
**Original author:** eterlan
**Created:** 2025-09-20

## Problem Summary

Ctrl/Cmd+Click ("Go to Definition") on built-in Godot type names (e.g. `ProjectSettings`, `String`, `int`, `float`, `Vector2i`) fails with errors. The extension cannot resolve the `gddoc:` URI scheme for these types. Hover tooltips work correctly — only navigation/opening the doc page is broken.

**IMPORTANT:** Navigation fails only for **SOME** built-in types. For `Node`, `Node2D` or `Input` it works just fine, for example.

## Error Chain

Three errors occur in sequence when trying to open a native type's documentation:

1. **`Unable to resolve resource gddoc:<Type>.gddoc`** — VS Code cannot resolve the custom `gddoc:` URI. This is the root cause.
2. **`Cannot read properties of undefined (reading 'body')`** — In the extension's code (`extension.js`), the `resolveCustomEditor` callback receives an undefined document (because step 1 failed) and tries to access `.body` on it.
3. **`Assertion Failed: Argument is 'undefined' or 'null'`** — VS Code's editor infrastructure fails because the custom editor returned nothing.

## Affected Types

Reporters have confirmed this happens with: `ProjectSettings`, `String`, `int`, `float`, `Vector2i`.

## Key Observations from Comments

### reuvenpo
- Affects both type names and their methods.
- Hover tooltips work correctly (only click-to-navigate is broken).
- Changing `godotTools.lsp.headless` has no effect.
- Points to `make_docs_uri` in `src/providers/documentation.ts` (line ~131) as the likely culprit — there's special handling for native classes that may be wrong.

### oppahansi
- Reproduced on **macOS Tahoe 26.2**, Godot 4.5.1, extension 2.5.1.
- Still broken on extension **2.6.0** (confirmed after Calinou asked them to test).
- Hover tooltip shows correct info; only Cmd+Click fails.

### Calinou (project member)
- Asked oppahansi to test v2.6.0 (the issue persisted).

### peterbozso (OP of this conversation context)
- Reproduced on **macOS**, Godot 4.6.1, extension **2.6.1** — still broken.
- Same `Unable to resolve resource gddoc:Vector2i.gddoc` error pattern.
- Also reports a **secondary issue**: Cmd+Click on `TileMapLayer.set_cell()` navigates to `TileMap`'s docs instead of `TileMapLayer`'s docs. The hover tooltip shows the correct class, but the wrong documentation page opens. This suggests the definition provider may be returning the wrong location for methods when a method name exists on multiple classes (especially parent/deprecated classes).

## Suspected Code Location

`src/providers/documentation.ts` — specifically the `make_docs_uri` function and the `resolveCustomEditor` handler. The URI resolution for native/built-in classes appears to be broken.

## Reproduction Steps

1. Open any Godot project in VS Code with the godot-tools extension.
2. Hover over some built-in types like `String`, `int`, or `Vector2i` — tooltip works fine.
3. Ctrl/Cmd+Click on the same type name — error occurs, no documentation opens.
