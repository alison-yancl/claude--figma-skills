# Claude Figma Skills

Three Claude Code skills that take a feature idea to polished Figma screens — without the tedious parts.

```
/design-spec  →  /figma-design  →  (you adjust in Figma)  →  /design-review
```

Each skill runs as a slash command inside Claude Code. They handle structuring specs, mining real sample data from existing demos, instantiating library components, and auditing token compliance — so you can focus on the design decisions that actually matter.

## Quickstart

```bash
git clone https://github.com/<you>/claude-figma-skills.git
cp -r claude-figma-skills/skills ~/.claude/skills
# or symlink into a project: ln -s "$PWD/claude-figma-skills/skills" .claude/skills
```

Then in Claude Code:

```
/design-spec  "Image search with filters for the Explore tab"
/figma-design
/design-review
```

## How it works

Each feature gets its own project folder in your working directory. Artifacts live there — **never inside `.claude/skills/`**.

```
<feature-name>/
  design-spec.md      ← from /design-spec
  design-review.md    ← from /design-review
```

---

## 1. `/design-spec` — Structured design brief

Turns a feature description into a `design-spec.md`: per-screen element tables, mock data, data-source mapping, and flow.

**Does**
- Searches the repo for related demos or prototypes and extracts real sample data (field names, content, edge cases) instead of generic placeholders.
- Reads API docs when provided — maps which fields each screen can display and which it can't.
- Produces per-screen **Elements** tables (component, content, data source) and **Mock Data** tables for dynamic content.

**You provide**
- Feature description (required)
- App / product (determines library + screen dimensions)
- Design library name
- API docs path (optional, but raises spec quality a lot)

**Won't**
- Decide screen count or flow on its own — it asks, you confirm.
- Invent data when there's no demo and no API docs — mock data falls back to generic.

---

## 2. `/figma-design` — Build screens in Figma

Reads the spec and builds screens in Figma using real library components, bound tokens, and proper text styles.

**Does**
- Enumerates the design library before building anything.
- Instantiates library components (not hand-drawn rectangles) wherever possible.
- Applies text styles and color variables from the library — no raw hex, no fallback fonts.
- Populates screens with mock data from the spec.
- Screenshots + self-reviews after building.

**You provide**
- Confirmation of the design library (asks even if `CLAUDE.md` names one)
- Target Figma file URL
- A reference file with existing screens if the target is empty (for token/pattern extraction)

**Won't**
- Make design decisions — it builds a starting point.
- Do prototyping, animation, or transitions.
- Work without a design library.

---

## 2.5 Iterate in Figma

The initial build is a starting point. Most refinement happens manually — layout tweaks, spacing, hierarchy, repositioning. That's where the design decisions live.

**Prompt Claude for changes that are tedious by hand:**
- Swapping variants across many screens
- Bulk text / mock data updates
- Rebinding tokens after manual edits break them
- Adding or removing repeated sections

**Tips**
- Reference screens by their Figma name.
- Be specific: *"On Search Results, change card layout from vertical to horizontal."*
- One change per prompt beats batching unrelated changes.
- Ask for a screenshot to verify.

**Watch out for** — manual edits can introduce issues `/design-review` will catch:
- **Detached components** from drag/restructure
- **Broken auto layout** from resizing
- **Unbound styles** from copy-paste (looks right, but fill is raw hex)

Move on to `/design-review` once layout feels right and only token/style cleanup remains.

---

## 3. `/design-review` — Audit and auto-fix

Audits every element for style compliance and fixes what it can unambiguously.

**Does**
- Builds lookup tables first — hex→variable map, font-signature→text-style map, from local + library sources.
- Audits every node outside component instances for:
  - Text without a text style
  - Solid fills without a bound color variable
  - Strokes without a bound color variable
  - Text with a text style but a raw hex fill (partial binding)
- **Auto-fixes exact matches only.** Same RGB → bind. Same font/size/weight → bind. No fuzzy matching.
- Writes a `design-review.md` report: before/after compliance, per-node tables, fixes applied, remaining issues.

**You provide**
- Figma URL or node IDs (or `"last"` to reuse the most recent screens from the spec)

**Won't**
- Create new tokens — flags unmatched colors, doesn't invent variables.
- Fix alignment or spacing — designer judgment.
- Audit gradient or image fills — solid only.

---

## Limitations

- **One library per feature.** Multi-library features need per-screen overrides.
- **Keyword-based component search.** Unconventional library naming can miss matches — you can point to specific components manually.
- **Mock data quality tracks inputs.** Demo > API docs > placeholder.
- **Conservative auto-fix.** Off-by-1 RGB gets flagged, not fixed.

## Repo layout

```
claude-figma-skills/
  skills/
    design-spec/SKILL.md
    figma-design/SKILL.md
    design-review/SKILL.md
  README.md
```

Install by copying or symlinking `skills/` into `~/.claude/skills/` (user scope) or `<project>/.claude/skills/` (project scope).
