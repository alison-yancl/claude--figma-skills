---
name: design-review
description: "Audit Figma screens for design quality — component coverage, token compliance, text styles, and visual consistency. Usage: /design-review [Figma URL or node IDs]"
argument-hint: "[Figma URL, node IDs, or 'last' to review most recent screens]"
---

# Design Review Skill

Audit built Figma screens for quality: library component usage, token compliance, text style binding, and visual consistency.

## Figma API Strategy (REST API — no Desktop Bridge needed)

All tools below use the Figma REST API. No need to open Figma Desktop or run the Desktop Bridge plugin.

| Tool | Purpose |
|------|---------|
| `mcp__figma__use_figma` | Execute Plugin API code (audit nodes, fix bindings). **Requires `fileKey` + `description` params.** |
| `mcp__figma__get_screenshot` | Visual validation screenshots. Requires `fileKey` + `nodeId`. |
| `mcp__figma__get_metadata` | Structural inspection (node tree, positions, sizes). Requires `fileKey` + `nodeId`. |
| `mcp__figma__search_design_system` | Search library components, variables, styles. Requires `fileKey` + `query`. |
| `mcp__figma__get_variable_defs` | Get resolved variable values for a node. Requires `fileKey` + `nodeId`. |

## Inputs

The user provides: `$ARGUMENTS`

This may be:
- A Figma URL (extract file key and node ID)
- Specific node IDs to review
- "last" — find the most recently created screens from the design spec

## Step 0: Auth Check (MANDATORY — run FIRST)

Before any Figma API calls, verify the `figma` MCP server is authenticated:

1. Make a lightweight test call: `mcp__figma__get_screenshot` with a known `fileKey` + `nodeId`
2. If it returns **"requires re-authorization (token expired)"** → STOP and tell the user:
   > "Figma MCP needs re-authorization. Please run `/mcp`, find the `figma` server, and complete the OAuth flow."
3. Do NOT proceed until the auth check passes
4. Do NOT suggest Desktop Bridge as a workaround — the skill uses REST API only

## Step 1: Identify Screens to Review

### If Figma URL provided:
Extract file key and node ID. Use `mcp__figma__get_metadata` (with `fileKey` + `nodeId`) to find all phone-sized frames (e.g., 390×844) under that node.

### If node IDs provided:
Use those directly.

### If "last" or empty:
Check for `design-spec.md` — first in any project subfolder (`<project-name>/design-spec.md`), then in the current directory. Extract the screen names and search for matching frames in the Figma file referenced in the spec.

## Step 2: Screenshot Each Screen

For each screen, use `mcp__figma__get_screenshot` (with `fileKey` + `nodeId`) to capture a visual snapshot. Review each screenshot for:
- [ ] Overall layout and alignment
- [ ] Visual hierarchy (headers, content, actions clearly ordered)
- [ ] Spacing consistency
- [ ] Dark/light theme consistency
- [ ] Text readability and contrast

Note any visual issues found.

## Step 3: Build Style & Variable Lookup Tables

Before auditing, collect all available styles and variables so you can match raw values to existing tokens.

**Quick option**: Use `mcp__figma__get_variable_defs` (with `fileKey` + `nodeId` of a screen) to get resolved variable values, and `mcp__figma__search_design_system` (with `includeVariables: true` and `includeStyles: true`) to find tokens by name.

**Full extraction** — run the following via `mcp__figma__use_figma` (with `fileKey`):

### 3a. Collect all color variables

```javascript
// Run via mcp__figma__use_figma with fileKey
// Build a hex → variable map from all local and library variables
const colorMap = {}; // key: "r,g,b,a" → value: { id, name, collection }

const collections = await figma.variables.getLocalVariableCollectionsAsync();
for (const col of collections) {
  for (const varId of col.variableIds) {
    const v = await figma.variables.getVariableByIdAsync(varId);
    if (v.resolvedType === 'COLOR') {
      // Get value for each mode
      for (const modeId of Object.keys(v.valuesByMode)) {
        const val = v.valuesByMode[modeId];
        if (val.r !== undefined) {
          const key = `${Math.round(val.r*255)},${Math.round(val.g*255)},${Math.round(val.b*255)}`;
          colorMap[key] = { id: v.id, name: v.name, collection: col.name };
        }
      }
    }
  }
}
```

### 3b. Collect all text styles

```javascript
// Run via mcp__figma__use_figma with fileKey
// Build a font signature → style map
const textStyleMap = {}; // key: "fontFamily-fontSize-fontWeight" → style id

const localStyles = await figma.getLocalTextStylesAsync();
for (const style of localStyles) {
  const key = `${style.fontName.family}-${style.fontSize}-${style.fontName.style}`;
  textStyleMap[key] = { id: style.id, name: style.name };
}

// Also collect library/remote styles from existing styled nodes
const styledNodes = screen.findAll(n => n.type === 'TEXT' && n.textStyleId);
for (const node of styledNodes) {
  const style = figma.getStyleById(node.textStyleId);
  if (style) {
    const key = `${node.fontName.family}-${node.fontSize}-${node.fontName.style}`;
    textStyleMap[key] = { id: style.id, name: style.name };
  }
}
```

### 3c. Also collect variable bindings from already-styled nodes

Walk nodes that ARE properly bound to expand the color map with library variables:

```javascript
// Run via mcp__figma__use_figma with fileKey (can combine with 3a/3b in one call)
const allNodes = screen.findAll(n => true);
for (const n of allNodes) {
  for (const prop of ['fills', 'strokes']) {
    if (n.boundVariables?.[prop]) {
      for (const binding of n.boundVariables[prop]) {
        const v = await figma.variables.getVariableByIdAsync(binding.id);
        if (v) {
          for (const modeId of Object.keys(v.valuesByMode)) {
            const val = v.valuesByMode[modeId];
            if (val.r !== undefined) {
              const key = `${Math.round(val.r*255)},${Math.round(val.g*255)},${Math.round(val.b*255)}`;
              colorMap[key] = { id: v.id, name: v.name, collection: 'library' };
            }
          }
        }
      }
    }
  }
}
```

## Step 4: Programmatic Audit

Use the lookup tables to find every unbound element. Run via `mcp__figma__use_figma` (with `fileKey`):

```javascript
// Run via mcp__figma__use_figma with fileKey
function isInsideInstance(node) {
  let parent = node.parent;
  while (parent) {
    if (parent.type === 'INSTANCE') return true;
    parent = parent.parent;
  }
  return false;
}

// 1. Component coverage
const instances = screen.findAll(n => n.type === 'INSTANCE');
const customFrames = screen.findAll(n =>
  n.type === 'FRAME' && n.parent?.type !== 'INSTANCE' && n.children?.length > 0
);
const coverage = instances.length / (instances.length + customFrames.length) * 100;

// 2. Unstyled text — no text style applied (outside instances)
const unstyledText = screen.findAll(n =>
  n.type === 'TEXT' && !n.textStyleId && !isInsideInstance(n)
);

// 3. Unbound fills — no color variable bound (outside instances)
const unboundFills = screen.findAll(n =>
  n.fills?.length > 0 &&
  n.fills.some(f => f.type === 'SOLID' && f.visible !== false) &&
  !n.boundVariables?.fills &&
  !isInsideInstance(n) &&
  n.type !== 'INSTANCE'
);

// 4. Unbound strokes — no color variable bound (outside instances)
const unboundStrokes = screen.findAll(n =>
  n.strokes?.length > 0 &&
  n.strokes.some(s => s.type === 'SOLID' && s.visible !== false) &&
  !n.boundVariables?.strokes &&
  !isInsideInstance(n) &&
  n.type !== 'INSTANCE'
);

// 5. Text with unbound fill color (text has style but color is raw hex)
const textUnboundColor = screen.findAll(n =>
  n.type === 'TEXT' &&
  n.textStyleId &&
  n.fills?.length > 0 &&
  !n.boundVariables?.fills &&
  !isInsideInstance(n)
);
```

## Step 5: Auto-Fix

For each issue found, attempt to fix by matching raw values against the lookup tables. **Only fix when the match is exact and unambiguous.** Run all fixes via `mcp__figma__use_figma` (with `fileKey`).

### 5a. Fix unstyled text → bind text style

```javascript
// Run via mcp__figma__use_figma with fileKey
for (const node of unstyledText) {
  await figma.loadFontAsync(node.fontName);
  const key = `${node.fontName.family}-${node.fontSize}-${node.fontName.style}`;
  const match = textStyleMap[key];
  if (match) {
    node.textStyleId = match.id;
    // Log: "Applied text style '{match.name}' to '{node.name}'"
  } else {
    // Log: "No matching text style for '{node.name}' ({key}) — needs manual fix"
  }
}
```

### 5b. Fix unbound fills → bind color variable

```javascript
// Run via mcp__figma__use_figma with fileKey
for (const node of [...unboundFills, ...textUnboundColor]) {
  for (let i = 0; i < node.fills.length; i++) {
    const fill = node.fills[i];
    if (fill.type === 'SOLID') {
      const key = `${Math.round(fill.color.r*255)},${Math.round(fill.color.g*255)},${Math.round(fill.color.b*255)}`;
      const match = colorMap[key];
      if (match) {
        node.setBoundVariable('fills', i, { type: 'VARIABLE_ALIAS', id: match.id });
        // Log: "Bound fill to '{match.name}' on '{node.name}'"
      } else {
        // Log: "No matching variable for fill rgb({key}) on '{node.name}' — needs manual fix"
      }
    }
  }
}
```

### 5c. Fix unbound strokes → bind color variable

```javascript
// Run via mcp__figma__use_figma with fileKey
for (const node of unboundStrokes) {
  for (let i = 0; i < node.strokes.length; i++) {
    const stroke = node.strokes[i];
    if (stroke.type === 'SOLID') {
      const key = `${Math.round(stroke.color.r*255)},${Math.round(stroke.color.g*255)},${Math.round(stroke.color.b*255)}`;
      const match = colorMap[key];
      if (match) {
        node.setBoundVariable('strokes', i, { type: 'VARIABLE_ALIAS', id: match.id });
        // Log: "Bound stroke to '{match.name}' on '{node.name}'"
      } else {
        // Log: "No matching variable for stroke rgb({key}) on '{node.name}' — needs manual fix"
      }
    }
  }
}
```

### 5d. Things NOT auto-fixed (flag for designer)

- **Misaligned elements** — needs designer judgment
- **Colors with no matching variable** — raw hex that doesn't exist in any token. Could be intentional or a mistake.
- **Text with no matching style** — font/size/weight combo not in the library. May need a new style or is wrong.
- **Gradient or image fills** — not bound to variables by convention, skip these

## Step 6: Generate Quality Report

Write the report to the project folder if a `design-spec.md` was found in one (e.g., `<project-name>/design-review.md`). Otherwise, write to `design-review.md` in the current working directory.

### Report Template:

```markdown
# Design Review: [Feature Name]

**Date**: [DATE]
**Screens reviewed**: [count]
**Figma file**: [file key]

## Summary

| Metric | Before | After Auto-Fix |
|--------|--------|----------------|
| Component coverage | [N%] | [N%] |
| Text style compliance | [N/M] nodes | [N/M] nodes |
| Fill color compliance | [N/M] nodes | [N/M] nodes |
| Stroke color compliance | [N/M] nodes | [N/M] nodes |

## Component Coverage

| Screen | Library Instances | Custom Elements | Coverage |
|--------|------------------|-----------------|----------|
| [Name] | [N] | [N] | [N%] |
| **Total** | **[N]** | **[N]** | **[N%]** |

Target: >80% ✓/✗

## Style Compliance (per node)

### Text Styles

| Node | Screen | Current Font | Matched Style | Status |
|------|--------|-------------|---------------|--------|
| [node name] | [screen] | [family size weight] | [style name] | ✓ Auto-fixed |
| [node name] | [screen] | [family size weight] | — | ✗ No match |

### Fill Colors

| Node | Screen | Raw Color | Matched Variable | Status |
|------|--------|-----------|-----------------|--------|
| [node name] | [screen] | rgb(r,g,b) | [variable name] | ✓ Auto-fixed |
| [node name] | [screen] | rgb(r,g,b) | — | ✗ No match |

### Stroke Colors

| Node | Screen | Raw Color | Matched Variable | Status |
|------|--------|-----------|-----------------|--------|
| [node name] | [screen] | rgb(r,g,b) | [variable name] | ✓ Auto-fixed |

## Visual Review

- [x/✗] Headers use library components
- [x/✗] Typography hierarchy is clear
- [x/✗] Colors follow design tokens
- [x/✗] Spacing is consistent
- [x/✗] Screen flow reads left-to-right
- [x/✗] Theme applied consistently

## Auto-Fixes Applied

| # | Node | Screen | What was fixed |
|---|------|--------|----------------|
| 1 | [name] | [screen] | Bound text style '[style name]' |
| 2 | [name] | [screen] | Bound fill to '[variable name]' |

Total: [N] fixes applied

## Remaining Issues (needs designer)

| # | Node | Screen | Issue | Why no auto-fix |
|---|------|--------|-------|-----------------|
| 1 | [name] | [screen] | Raw fill rgb(X,X,X) | No matching variable found |
| 2 | [name] | [screen] | Unstyled text ([font info]) | No matching text style — may need new style or wrong font |

## Screenshots
[Reference each screen screenshot taken during review]
```

## Rules

- **Screenshot every screen** — visual review is mandatory, not optional
- **Build lookup tables first** — always collect available variables and text styles before auditing. You can't match what you don't know about.
- **Check fills, strokes, AND text colors** — a node can have a text style applied but still use a raw hex fill color. Check both independently.
- **Don't silently pass** — if unstyled text or unbound colors exist, report them even if coverage is >80%
- **Auto-fix when exact match exists** — if a raw hex value matches a known variable exactly (same RGB), bind it. If a font/size/weight matches a known text style exactly, bind it. No fuzzy matching.
- **Report before-and-after** — the summary table should show compliance before and after auto-fix so the designer sees the impact
- **Be specific** — name the exact node, its screen, its current raw value, and what it should be
- **No false positives** — nodes inside component instances inherit styles from the component; don't flag those
- **Skip gradients and image fills** — only audit SOLID fills and strokes
