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

1. **`Unable to resolve resource gddoc:<Type>.gddoc`** — VS Code surfaces this error when `resolveCustomEditor` throws an unhandled exception during the doc page render.
2. **`Cannot read properties of undefined (reading 'body')`** — The actual crash site. `make_symbol_elements()` in `documentation_builder.ts` returns `undefined` for children it can't handle, and the caller accesses `.body` on the result without a null check. (See Root Cause Analysis below.)
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

## Reproduction Steps

1. Open any Godot project in VS Code with the godot-tools extension.
2. Hover over some built-in types like `String`, `int`, or `Vector2i` — tooltip works fine.
3. Ctrl/Cmd+Click on the same type name — error occurs, no documentation opens.

---

## Root Cause Analysis

### How Go to Definition works (the full flow)

1. User Ctrl/Cmd+Clicks on a symbol in a `.gd` file.
2. `GDDefinitionProvider.provideDefinition()` (`src/providers/definition.ts:31`) calls `get_symbol_at_position()`.
3. `get_symbol_at_position()` (`src/lsp/GDScriptLanguageClient.ts:310`) sends a `textDocument/hover` request to Godot's LSP.
4. Godot's LSP resolves the symbol via `resolve_symbol()` → `lookup_code()` → `get_native_symbol()` and returns hover text like `"\t<Native> class int\n\n..."` or `"\tfunc Node.add_child(...) -> void\n\n..."`.
5. `parse_hover_result()` (`GDScriptLanguageClient.ts:322`) extracts the class/member name using two regexes:
   - `/(?:func|const) (@?\w+)\.(\w+)/` → matches methods/constants, returns `"ClassName.memberName"`
   - `/<Native> class (\w+)/` → matches native class names, returns `"ClassName"`
6. `provideDefinition()` splits the result on `.` and calls `make_docs_uri(className, memberName)` to create a `gddoc:ClassName.gddoc#memberName` URI.
7. VS Code opens this URI, routed to `GDDocumentationProvider.resolveCustomEditor()` (`src/providers/documentation.ts:81`).
8. `resolveCustomEditor()` sends a `textDocument/nativeSymbol` request to Godot's LSP to fetch the full class symbol tree (methods, properties, constants, signals, constructors, operators).
9. `make_html_content()` (`src/providers/documentation_builder.ts:58`) renders the symbol tree into an HTML webview by calling `make_symbol_document()`.
10. `make_symbol_document()` iterates `symbol.children`, calling `make_symbol_elements()` for each child to produce the HTML.

### Bug 1: `make_symbol_elements()` crashes on Constructor and Operator children

**Crash site:** `src/providers/documentation_builder.ts:237–259`

`make_symbol_elements()` (line 139) has a switch on `s.kind` that handles: `Property` (7), `Variable` (13), `Constant` (14), `Event` (24), `Method` (6), `Function` (12). For any other kind, it falls through to `default: break` and **returns `undefined`**.

Godot's LSP assigns `SymbolKind.Constructor` (9) and `SymbolKind.Operator` (25) to constructor and operator children respectively (set in `gdscript_workspace.cpp:310–317`). These are valid LSP 3.x `SymbolKind` values, but the extension's `make_symbol_elements()` has no case for them.

The calling code in `make_symbol_document()` (line 237) iterates children and uses the return value:

```typescript
const elements = make_symbol_elements(s);
switch (s.kind) {
    // ... handled cases ...
    default:
        others += element("li", elements.body, { id: s.name }); // CRASH: elements is undefined
}
```

When `elements` is `undefined`, accessing `elements.body` throws `TypeError: Cannot read properties of undefined (reading 'body')` — this is **Error #2** from the issue.

**Why this only affects Variant types:** Object-derived engine classes (`Node`, `Node2D`, `Input`, etc.) only have children with kinds `Property`, `Constant`, `Method`, and `Event` (signals). They have **no constructors or operators** in their documentation. Variant built-in types (`int`, `float`, `String`, `Vector2i`, `bool`, `Color`, `Array`, `Dictionary`, etc.) have both **constructors** (e.g., `int(from: float)`) and **operators** (e.g., `int + int`), so they always trigger this crash.

**Godot engine side (for reference):** Constructors and operators are processed in `gdscript_workspace.cpp:296–360` in a unified loop alongside methods and signals. The kind assignment is:
```
i >= signal_start_idx      → SymbolKind::Event (24)
i >= operator_start_idx    → SymbolKind::Operator (25)
i >= constructors_start_idx → SymbolKind::Constructor (9)
else                       → SymbolKind::Method (6)
```
All four groups share the same `detail` format: `"func ClassName.name(args) -> RetType"`. So Constructor and Operator children CAN be rendered by the existing method-rendering code — they just need to be routed there.

### Bug 2: Property name regex too restrictive for `ProjectSettings`

**Crash site:** `src/providers/documentation_builder.ts:144`

`make_symbol_elements()` for `Property`/`Variable` kinds parses the `detail` string with:
```typescript
const parts = /\.([A-z_0-9]+)\:\s(.*)$/.exec(s.detail);
if (!parts) { return; } // returns undefined!
```

`ProjectSettings` has hundreds of properties with `/` in their names (e.g., `application/boot_splash/bg_color`, `display/window/size/viewport_width`). These come from the engine's `ProjectSettings.xml` doc file, where each engine setting is documented as a class member. The LSP produces detail strings like:
```
var ProjectSettings.application/boot_splash/bg_color: Color
```

The regex character class `[A-z_0-9]` does **not** match `/` (ASCII 47 is outside the `A`=65 to `z`=122 range). So `parts` is `null`, the function returns `undefined`, and the caller crashes on `elements.index` (line 242) with a similar TypeError.

### Both bugs share the same underlying pattern

`make_symbol_elements()` can return `undefined` in two ways:
1. Unhandled `SymbolKind` (Constructor, Operator) → `default: break` with no return value
2. Failed regex match on property names with special characters → bare `return;`

The calling code in `make_symbol_document()` (lines 237–259) never checks the return value before accessing `.body` or `.index`. The first unhandled child encountered causes the entire `resolveCustomEditor()` to throw, which VS Code surfaces as "Unable to resolve resource gddoc:X.gddoc".

### Why types that work are unaffected

Types like `Node`, `Node2D`, `Input`:
- Have no Constructor or Operator children (they're Object-derived, not Variant types)
- Have property names using only word characters (e.g., `position`, `visible`, `name`)
- All their children are handled by existing `make_symbol_elements()` cases

---

## TileMapLayer/TileMap Navigation Issue

### Analysis

`TileMapLayer` and `TileMap` are **sibling classes** — both extend `Node2D` directly (`tile_map_layer.h:332`, `tile_map.h:51`). There is **no inheritance relationship** between them. Each defines its own `set_cell()` method with different signatures:
- `TileMapLayer.set_cell(coords: Vector2i, source_id: int, atlas_coords: Vector2i, alternative_tile: int)`
- `TileMap.set_cell(layer: int, coords: Vector2i, source_id: int, atlas_coords: Vector2i, alternative_tile: int)`

Godot's LSP resolves method lookups through `_lookup_symbol_from_base()` (`gdscript_editor.cpp:4103`), which calls `ClassDB::has_method(class_name, "set_cell", true)` with `p_no_inheritance = true`. For a variable typed as `TileMapLayer`, it starts with `class_name = "TileMapLayer"` and finds `set_cell` on the first iteration. So `r_result.class_name` should be `"TileMapLayer"`, and the hover text should read `func TileMapLayer.set_cell(...)`.

The `parse_hover_result()` regex `/(?:func|const) (@?\w+)\.(\w+)/` would correctly extract `TileMapLayer` as `match[1]`, producing the correct gddoc URI.

### Possible explanations for the reported misbehavior

1. **Untyped variable:** If the variable is declared as `var layer = $TileMapLayer` without a type annotation, and type inference fails or resolves to a parent type, the method lookup could start from a different class.
2. **Deprecated API confusion:** `TileMap` is deprecated since Godot 4.3 in favor of `TileMapLayer`. If the user actually had a `TileMap` variable, navigation to `TileMap`'s docs would be correct behavior.
3. **Version-specific issue:** In transitional Godot versions, `TileMapLayer` might not have had `set_cell` registered in `ClassDB` yet, causing the method lookup to fail entirely and fall back to smart_resolve, which could match `TileMap.set_cell` first.
4. **Smart resolve fallback:** If `resolve_symbol()` returns `nullptr` (because `lookup_code` fails), Godot's hover handler falls back to `resolve_related_symbols()` which does a global fuzzy search across ALL native members. If `TileMap.set_cell` appears before `TileMapLayer.set_cell` in this search, it would be returned first. The response in this case is an array (not a MarkupContent object), and `parse_hover_result()` takes `contents[0]`, which would be the first match.

### Relationship to Bug 1

The TileMap/TileMapLayer issue is **independent** of the gddoc crash bug. They have entirely different root causes:
- **Bug 1** (gddoc crash): Extension-side rendering crash in `documentation_builder.ts` — unhandled SymbolKinds and restrictive property name regex.
- **TileMap issue**: LSP-side symbol resolution returning the wrong class for a method call, possibly due to smart_resolve fallback or type inference failure.

Fixing Bug 1 will NOT fix the TileMap issue, and vice versa. They can be addressed independently. However, once Bug 1 is fixed, the TileMap issue may become easier to diagnose because the docs page for `TileMapLayer` will actually open (currently it might also crash due to Bug 1 if `TileMapLayer` has any children with unhandled kinds).

---

## Fix Strategy

### For Bug 1 (Crash on Constructor/Operator children)

**Primary fix in `src/providers/documentation_builder.ts`:**
1. Add `SymbolKind.Constructor` (9) and `SymbolKind.Operator` (25) cases to both the `make_symbol_elements()` switch and the main loop in `make_symbol_document()`. Since their `detail` format is identical to methods (`"func ClassName.name(args) -> RetType"`), they can reuse the existing method-rendering code.
2. Add a null guard on the `make_symbol_elements()` return value in the main loop (defensive fix).

### For Bug 2 (ProjectSettings property regex)

**Fix in `src/providers/documentation_builder.ts`:**
1. Broaden the property name regex from `[A-z_0-9]+` to also match `/` (and potentially other special characters that appear in Godot property names). For example: `[A-Za-z_0-9/]+` or `[^:]+` (match everything up to the colon).

### For the TileMap/TileMapLayer issue

Further investigation is needed. Potential approaches:
1. **Reproduce with debugging:** Set up a test project with a typed `TileMapLayer` variable and log the actual hover response to confirm whether the LSP returns the correct class.
2. **If smart_resolve is the culprit:** The fix would be Godot-engine-side, in how `resolve_related_symbols()` prioritizes matches when the same method name exists on multiple classes.
3. **If the definition provider should use `textDocument/definition` instead of `textDocument/hover`:** This would be a larger refactor of `GDDefinitionProvider` but would be more correct architecturally, as the hover hack loses information that the LSP's definition response preserves.
