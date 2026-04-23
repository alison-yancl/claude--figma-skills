---
name: design-spec
description: "Create a structured design spec for a Figma feature — screens, elements, mock content, data mapping, and flow. Discovers existing demos/prototypes for real sample data. Outputs to a project folder. Usage: /design-spec [feature description]"
argument-hint: "[feature description, optionally with API doc path or demo path]"
---

# Design Spec Skill

Create a structured design brief for Figma — what screens to design, what elements they contain, what mock content to use, what data is available, and how they flow together.

## Inputs

The user provides: `$ARGUMENTS`

This may include:
- A feature description (required)
- A path to API docs or reference files in the repo (optional)
- A path to an existing demo or prototype (optional)
- A Figma file URL or key (optional)

## Step 0: Determine Project Folder

All design artifacts live in a **project folder** — one folder per feature.

### 0a. Infer the project name
Extract a short kebab-case name from the feature description (e.g., "Image search feature" → `image-search`).

### 0b. Find or create the project folder
Check if a matching folder already exists in the current working directory:
```
./<project-name>/
```

If it exists, use it. If not, create it. All outputs (`design-spec.md`, and later `design-review.md`, etc.) go into this folder.

### 0c. Check for existing spec
If `<project-name>/design-spec.md` already exists, ask:
> "A design spec already exists for this project. Should I (a) update it, or (b) start fresh?"

## Step 1: Gather Context

### 1a. Parse the feature description
Extract: feature name, target screens, user flow, and any mentioned constraints.

### 1b. Check for existing demo or prototype
Search the repo for an existing prototype or demo related to this feature:
- Check common prototype directories (e.g., `/prototypes/`, `/playground/`, `/examples/`) for matching directories (fuzzy match on feature name)
- Check paths explicitly mentioned in `$ARGUMENTS`

If a demo/prototype is found:
- Read its code, sample data, test fixtures, and any README/docs
- Extract: real field names, data shapes, content examples, edge cases, user flows
- Use its sample data as the basis for **Mock Data** in the spec (real content > placeholder content)
- Note any states or error handling the demo already covers
- Record the demo path in the spec header as **Demo source**

If no demo found, proceed — the spec will use placeholder mock data instead.

### 1c. Check for API docs
If the user provided a file path or mentions API docs:
- Read the file(s)
- Extract available data fields, response shapes, and limitations
- Note what data IS and IS NOT available (this constrains the design)

If no API docs provided, ask ONE question:
> "What data/info is available for this feature? (e.g., product has: title, price, image, seller) — or say 'skip' if not applicable."

### 1d. Confirm the app/product context
Ask the user to confirm which app or product this feature belongs to. This determines:
- Which design library to reference
- Screen dimensions (e.g., 390×844 for iOS app, 1440×900 for web)
- Which existing Figma files can serve as reference for patterns and styles

If the user has already stated the app context clearly, proceed without asking.

### 1e. Check for design library info
Look in the current directory and parent CLAUDE.md files for:
- Design library file key
- Target Figma file key
- Existing component catalog or token reference

If no library info is found, ask:
> "Which Figma design library should I reference? Please share the library URL or file key."

## Step 2: Generate Design Spec

Write to `<project-name>/design-spec.md`.

### Template:

```markdown
# Design Spec: [Feature Name]

**Created**: [DATE]
**Figma file**: [file key or URL if known]
**Design library**: [library key if known]
**Demo source**: [path to demo/prototype, or "none"]

## Flow

[Screen 1] → [Screen 2] → [Screen 3]

## Screens

### Screen 1: [Name]

- **Purpose**: [What the user does here — one sentence]
- **States**: [Default, Loading, Empty, Error — only include relevant ones]

#### Elements

| Element | Component | Content | Data Source | Notes |
|---------|-----------|---------|-------------|-------|
| [name] | [library component or "custom"] | [static text or "see Mock Data"] | [API field path or "static"] | [max length, format, etc.] |

#### Mock Data

> Include when the screen displays dynamic/repeated content (lists, cards, grids).
> Skip for screens with only static content.

| Field | Sample 1 | Sample 2 | Sample 3 |
|-------|----------|----------|----------|
| [field] | [value] | [value] | [value] |

### Screen 2: [Name]
...

[Repeat for each screen]

## Data Mapping (from API)

> Skip this section entirely if no API docs were provided.

| Screen | Field | Source | Format | Constraints |
|--------|-------|--------|--------|-------------|
| [Screen name] | [Display field] | [API field path] | [string, number, URL, etc.] | [max length, nullable, etc.] |

## Design Constraints

> Skip if no constraints identified.

- [What data is NOT available — affects what can/can't be shown]
- [API limitations — max results, rate limits, etc.]
- [Platform constraints — iOS only, specific screen sizes, etc.]

## Design Library

- **Library**: [name and file key]
- **Target file**: [name and file key]
- **Theme**: [Dark/Light]
- **Screen size**: [e.g., 390×844 iOS]
```

## Rules

- **Structured elements** — every screen uses an Elements table, not free-form bullet lists. This makes the spec parseable by `/figma-design`.
- **Mock data from demos first** — if a demo/prototype exists, extract real sample data from it. Only use placeholder data as a fallback.
- **Data-aware** — if API docs exist, the spec should clearly show what info each screen can display.
- **Constraint-forward** — surface what CAN'T be done early, so the designer doesn't design for unavailable data.
- **Skip empty sections** — if no API docs, drop "Data Mapping" and "Design Constraints" entirely. If no dynamic content, drop "Mock Data" for that screen.
- **Project folder** — always output to the project folder, never to the root working directory.
- **Keep it scannable** — structured ≠ verbose. Tables should be concise. One page per screen max.

## After Generation

Tell the user:
> Design spec ready at `<project-name>/design-spec.md`. Next step: run `/figma-design` to start building, or edit the spec first.
