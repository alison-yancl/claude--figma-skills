---
name: figma-design
description: "Create or edit Figma designs using the project's design library components, styles, and tokens. Enforces component-first design, proper style application, and post-creation review. Usage: /figma-design [description of what to design]"
argument-hint: "[description of what to design]"
---

# Figma Design Skill

Create or edit Figma designs using the project's design library — component-first, style-bound, and review-verified.

## Inputs

The user provides: `$ARGUMENTS`

### Check for Design Spec

Look for a design spec in order:
1. `<project-name>/design-spec.md`
2. `./design-spec.md`
3. `./<feature-name>/design-spec.md`

If found, use it as the primary input (screens, elements, mock data, flow). If not, proceed with `$ARGUMENTS`.

### Figma API Strategy (REST API — no Desktop Bridge needed)

All tools below use the Figma REST API. No need to open Figma Desktop or run the Desktop Bridge plugin.

| Tool | Purpose |
|------|---------|
| `mcp__figma__use_figma` | Execute Plugin API code (create, edit, inspect nodes). **Requires `fileKey` param.** |
| `mcp__figma__get_screenshot` | Visual validation screenshots. Requires `fileKey` + `nodeId`. |
| `mcp__figma__get_metadata` | Structural inspection (node tree, positions, sizes). Requires `fileKey` + `nodeId`. |
| `mcp__figma__get_design_context` | Full design context (code + screenshot + hints). Requires `fileKey` + `nodeId`. |
| `mcp__figma__search_design_system` | Search library components, variables, styles by name. Requires `fileKey` + `query`. |
| `mcp__figma__get_variable_defs` | Get resolved variable values for a node. Requires `fileKey` + `nodeId`. |

**Important**: Every `use_figma` call must include the `fileKey` parameter and a `description` string.

## Phase 0: Auth Check (MANDATORY — run FIRST)

Before any Figma API calls, verify the `figma` MCP server is authenticated:

1. Make a lightweight test call: `mcp__figma__get_screenshot` with a known `fileKey` + `nodeId`
2. If it returns **"requires re-authorization (token expired)"** → STOP and tell the user:
   > "Figma MCP needs re-authorization. Please run `/mcp`, find the `figma` server, and complete the OAuth flow."
3. Do NOT proceed to Phase 1 until the auth check passes
4. Do NOT suggest Desktop Bridge as a workaround — the skill uses REST API only

## Phase 1: Context Gathering

Run in parallel where possible.

### 1a. Confirm the Design Library (MANDATORY)

Always ask: *"Which design library should I use? Please share the library URL or file key."*

### 1b. Find the Target Figma File

Extract file key and node ID from user's URL, or check CLAUDE.md.

### 1c. Understand Existing Screens

Use `mcp__figma__get_metadata` and `mcp__figma__get_screenshot` (both with `fileKey` + `nodeId`) to understand existing screens, components in use, and layout conventions.

### 1d. Empty File Handling

If the target file has no existing screens:
1. Ask for a reference file with existing screens
2. Extract tokens, text styles, and component patterns from 2-3 reference screens
3. Verify screen dimensions (e.g., 390x844 for iOS, 1440x900 for web)

## Phase 2: Component Audit

**Never hand-draw what already exists — not even small elements like badges, dividers, or tip bars.**

### 2a. Full Component Inventory (MANDATORY — run FIRST)

Scan the entire target page for all unique library component instances using `mcp__figma__use_figma`:

```javascript
// Run via mcp__figma__use_figma with fileKey
const page = figma.currentPage;
const allInstances = page.findAll(n => n.type === "INSTANCE");
const uniqueComponents = new Map();
for (const inst of allInstances) {
  try {
    const mc = await inst.getMainComponentAsync();
    if (!mc) continue;
    const compSet = mc.parent?.type === "COMPONENT_SET" ? mc.parent : null;
    const setKey = compSet ? compSet.key : mc.key;
    if (!uniqueComponents.has(setKey)) {
      uniqueComponents.set(setKey, {
        setName: compSet ? compSet.name : mc.name,
        setKey, variantName: mc.name, variantKey: mc.key,
        size: `${Math.round(inst.width)}x${Math.round(inst.height)}`,
        count: 1,
      });
    } else { uniqueComponents.get(setKey).count++; }
  } catch(e) {}
}
return Object.fromEntries(uniqueComponents);
```

This reveals all components including small ones (badges, indicators, labels, field messages, dividers, toasts).

**Note**: In REST API mode, use `await inst.getMainComponentAsync()` instead of `inst.mainComponent` (sync property may not be available).

### 2b. Search the Design Library (supplement)

Use `mcp__figma__search_design_system` with the `fileKey` to find components not yet in the file:

```
search_design_system({ query: "Button", fileKey: "xxx", includeComponents: true })
```

This searches across all published libraries linked to the file. If it fails, the inventory from 2a is sufficient.

### 2c. Component Match Verification

Map every design element to either a library component or explicitly mark as "custom" with justification. **Do NOT proceed to Phase 4 until complete.**

### 2d. Match Existing Patterns

Use the same components and variants that existing screens already use.

### 2e. Catalog in CLAUDE.md

If no component catalog exists, create one with: library file key, component key table, and design tokens.

## Phase 3: Extract Design Tokens (HARD GATE)

**Do NOT proceed to Phase 4 until `varMap` and `styleMap` are populated.**

### 3a. Build Variable ID Lookup Table

Extract actual Variable IDs from existing screens — required for `setBoundVariable()`.

**Quick option**: Use `mcp__figma__get_variable_defs` with `fileKey` + `nodeId` of an existing screen to get resolved variable names and values without running code.

**Full extraction** via `mcp__figma__use_figma`:

```javascript
// Run via mcp__figma__use_figma with fileKey
const existingScreen = figma.getNodeById("NODE_ID_HERE");
const varMap = {};
for (const n of existingScreen.findAll(n => true)) {
  for (const prop of ['fills', 'strokes']) {
    if (n.boundVariables?.[prop]) {
      for (const binding of n.boundVariables[prop]) {
        const v = await figma.variables.getVariableByIdAsync(binding.id);
        if (v) varMap[v.name] = binding.id;
      }
    }
  }
}
return varMap;
// Output: { "background/bg-primary": "VariableID:xxx", "text/text-primary": "VariableID:yyy", ... }
```

**Tip**: Also use `mcp__figma__search_design_system` with `includeVariables: true` to find variables by name (e.g., `query: "bg-sheet"`).

### 3b. Build Text Style Lookup Table

Extract text style keys from existing screens — required for `importStyleByKeyAsync()`.

**Quick option**: Use `mcp__figma__search_design_system` with `includeStyles: true` to find styles by name (e.g., `query: "Body 2"`).

**Full extraction** via `mcp__figma__use_figma`:

```javascript
// Run via mcp__figma__use_figma with fileKey
const existingScreen = figma.getNodeById("NODE_ID_HERE");
const styleMap = {};
for (const t of existingScreen.findAll(n => n.type === "TEXT")) {
  if (t.textStyleId) {
    const style = figma.getStyleById(t.textStyleId);
    if (style) styleMap[style.name] = { id: t.textStyleId, key: style.key };
  }
}
return styleMap;
// Output: { "Body 2/Regular": { id: "S:xxx", key: "abc123" }, ... }
```

### 3c. Verify Tables

- `varMap`: must have core tokens like `background/bg-primary`, `text/text-primary`, `text/text-secondary`, `border/border-primary`, etc.
- `styleMap`: must have core text styles like `Body/Regular`, `Body/Medium`, `Subtitle/Bold`, etc.

If empty, scan more screens or ask for a reference file.

## Phase 4: Build the Design

### 4a. Screen Structure

Build top-to-bottom: Status bar → Header → Content → Bottom bar / Home Indicator. All from library components.

### 4b. Component Instantiation

- Use `figma.importComponentByKeyAsync(variantKey)` then `.createInstance()`
- Use **variant keys** (not component set keys)
- Load fonts before setting text: `await figma.loadFontAsync(textNode.fontName)`

### 4c. Component Import Fallback

1. Try `importComponentByKeyAsync` → `.createInstance()`
2. If fails → find existing instance on page → `.clone()`
3. Never hand-draw without documenting why

### 4d. Slot Content

Build content completely before `slot.appendChild()` — node references become stale after insertion. Set all properties, sizes, and text before appending. Use separate `use_figma` calls per insertion.

### 4e. Style Application (HARD GATE)

**Every element must use `varMap` and `styleMap` from Phase 3.**

#### Text nodes

```javascript
async function createStyledText(content, styleName, colorVarName) {
  const t = figma.createText();
  const style = await figma.importStyleByKeyAsync(styleMap[styleName].key);
  t.textStyleId = style.id;
  t.setBoundVariable('fills', 0, { type: "VARIABLE_ALIAS", id: varMap[colorVarName] });
  t.characters = content;
  return t;
}
```

Never use `fontName = { family: ... }` or `fills = [{ type: "SOLID", color: { r:... } }]` for text.

#### Frames and shapes

```javascript
frame.setBoundVariable('fills', 0, { type: "VARIABLE_ALIAS", id: varMap["background/bg-sheet"] });
frame.setBoundVariable('strokes', 0, { type: "VARIABLE_ALIAS", id: varMap["border/border-primary"] });
```

Never use `fills = [{ type: "SOLID", color: { r:... } }]` for frames.

#### Exceptions

For gradients, image fills, or decorative elements with no matching variable: use raw values with a comment `// No variable — decorative` and report in Phase 5 audit.

#### Self-check before each `use_figma` call

Scan code for `{ r:`, `fontName = {`, `fontSize =`, or `fills = [{ type: "SOLID"`. Fix before executing.

### 4f. Auto Layout (MANDATORY)

Every multi-child frame must use auto layout. Verify after building:

```javascript
const noAutoLayout = screen.findAll(n =>
  n.type === 'FRAME' && n.children?.length > 1 &&
  n.layoutMode === 'NONE' && !isInsideInstance(n)
);
```

## Phase 5: Post-Creation Review

### 5a. Visual Review

Screenshot each screen using `mcp__figma__get_screenshot` (with `fileKey` + screen `nodeId`) and check: library Header, text styles applied, color variables bound, components used, spacing consistent, alignment correct.

### 5b. Style Binding Audit

Run via `mcp__figma__use_figma` with `fileKey`:

```javascript
const screen = figma.getNodeById("SCREEN_NODE_ID");
function isInsideInstance(node) {
  let p = node.parent;
  while (p) { if (p.type === 'INSTANCE') return true; p = p.parent; }
  return false;
}
const unstyledText = screen.findAll(n =>
  n.type === "TEXT" && !n.textStyleId && !isInsideInstance(n)
);
const unboundFills = screen.findAll(n =>
  n.fills?.length > 0 && !n.boundVariables?.fills && !isInsideInstance(n)
);
return {
  unstyledTextCount: unstyledText.length,
  unstyledText: unstyledText.map(n => ({ id: n.id, name: n.name })),
  unboundFillsCount: unboundFills.length,
  unboundFills: unboundFills.map(n => ({ id: n.id, name: n.name })),
};
```

### 5c. Component Coverage

Report library instances vs custom elements. **Hard gate**: below 80% coverage → stop and fix.

### 5d. Fix Issues

Fix all issues before presenting to the user.

## Phase 6: Organize & Label

- Flow section label (using an accent color variable from the library)
- Screen sublabels (using `text/text-secondary` or equivalent)
- Flow arrows between sequential screens
- 450px gaps between screens

## Checklist

| # | Check | Required |
|---|-------|----------|
| 1 | Design library confirmed | Yes |
| 2 | Full component inventory completed (Phase 2a) | Yes |
| 3 | Every element mapped to library component or marked custom | Yes |
| 4 | Library components used (>=80% coverage) | Yes |
| 5 | `varMap` populated with Variable IDs (Phase 3a) | Yes |
| 6 | `styleMap` populated with text style keys (Phase 3b) | Yes |
| 7 | All text uses `importStyleByKeyAsync` — no manual `fontName` | Yes |
| 8 | All fills use `setBoundVariable` — no raw hex | Yes |
| 9 | All multi-child frames use auto layout | Yes |
| 10 | Screenshots reviewed | Yes |
| 11 | Style binding audit passed (no unstyled text or unbound fills) | Yes |
| 12 | Status bar, Header, Home Indicator are library instances | Yes |
| 13 | CLAUDE.md updated with component catalog | Yes |
| 14 | Flow labels and organization added | Yes |
