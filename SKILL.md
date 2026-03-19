---
name: code-audit
description: Extract and audit a design system from any codebase. Use when the user invokes /code-audit, asks to "audit a codebase", "extract design tokens", "audit the design system", or wants to compare design implementations across platforms or against a Figma file. Works with any tech stack — web (CSS/SCSS/Tailwind), iOS (Swift/UIKit/SwiftUI), Android (XML/Compose), React Native, Flutter, etc.
---

# Code Audit — Design System Extraction & Analysis

Extracts design tokens (colors, typography, spacing, corner radius, elevation, icons) from
any codebase, assesses maturity, identifies inconsistencies, and optionally compares across
platforms or against a Figma design file.

---

## When This Skill Activates

- `/code-audit [path]` — explicit invocation
- "audit the design system in ..."
- "extract design tokens from ..."
- "compare iOS and Android design systems"
- "how does the code match the Figma?"
- "what tokens does this codebase use?"

---

## Step 0 — Scope & Orientation

Ask the user (if not already clear):

1. **Codebase path(s)** — one or more local directories to audit
2. **Platforms** — web, iOS, Android, React Native, Flutter, etc. (auto-detect if not specified)
3. **Focus** — full audit, or specific categories (colors only, typography only, etc.)
4. **Comparison mode** — single codebase, cross-platform, or code-vs-Figma
5. **Component focus** — full system audit, or drill into a specific component (e.g., Button)
6. **Output directory** — where to write audit artifacts (default: `audit/` inside the repo, or a user-specified path)

If the user provides a repo URL instead of a local path, clone it (shallow: `git clone --depth 1`) first.

---

## Step 1 — Detect Tech Stack

Scan the codebase to identify the UI technology and token sources. Use the table below as a starting point — adapt to what you actually find.

| Stack | Token locations to search | File patterns |
|-------|--------------------------|---------------|
| **Web (CSS/SCSS)** | CSS custom properties, SCSS variables, Tailwind config | `*.css`, `*.scss`, `tailwind.config.*` |
| **Web (CSS-in-JS)** | Theme objects, styled-components, Emotion | `theme.ts`, `*.styled.ts`, `tokens.*` |
| **Web (Tailwind)** | Tailwind config, `@apply` directives, utility classes | `tailwind.config.*`, `*.css` with `@apply` |
| **React Native** | StyleSheet objects, theme providers | `*.styles.ts`, `theme.*`, `colors.*` |
| **iOS (UIKit)** | UIColor extensions, Asset catalogs, plists | `*.swift`, `*.xcassets`, `Colors.xcassets` |
| **iOS (SwiftUI)** | Color extensions, Asset catalogs | `*.swift`, `*.xcassets` |
| **Android (XML)** | Resource files | `res/values/colors.xml`, `res/values/dimens.xml`, `res/values/styles.xml`, `res/values/themes.xml` |
| **Android (Compose)** | Kotlin theme/color objects | `Theme.kt`, `Color.kt`, `Type.kt`, `Shape.kt` |
| **Flutter** | ThemeData, ColorScheme | `theme.dart`, `colors.dart`, `text_theme.dart` |
| **Design tokens** | W3C/Style Dictionary format | `tokens.json`, `*.tokens.json`, `style-dictionary.config.*` |

**Auto-detection strategy:**
1. Check for framework config files (`package.json`, `Podfile`, `build.gradle`, `pubspec.yaml`)
2. Glob for the file patterns above
3. Report what was found before proceeding

### CSS-in-JS specifics (styled-components, Emotion, Stitches, etc.)

When the stack is CSS-in-JS:
- **Theme file is the primary token source.** Read it first — it replaces `tokens.css` / `colors.xml` as the system of record.
- **Search `.ts` and `.tsx` files** for token usage, not `.css` files.
- **Look for the TypeScript type definition** (e.g., `styled-components.d.ts`, `DefaultTheme` interface) — this is the authoritative list of every theme property.
- **Color manipulation libraries** (polished, color2k, chroma-js) generate computed values. When you encounter `darken()`, `lighten()`, `transparentize()`, etc.:
  - Note the base color + function call (e.g., `darken(0.15, '#FFFFFF')`)
  - **Resolve exact values by running the library.** Install the exact version from the project's package.json into a temp directory (`cd /tmp && npm init -y && npm install polished@<version>`) and run a Node script to compute every expression. Manual approximation of HSL math is unreliable — verified diffs of 3–8 hex digits between guessed and actual values (e.g., `darken(0.05, #ed2651)` guessed as `#D9204A`, actual `#e61341`).
  - If the library cannot be installed (e.g., internal/proprietary), note the expression as-is and flag values as "unverified approximation" in the audit.
  - Include both the expression AND the resolved value in all output (JSON, HTML, reference tables) so designers have the exact color and the derivation rule.
- **Theme builder patterns** (e.g., `buildLightTheme()` / `buildDarkTheme()`) indicate a layered architecture. Trace the builder chain to map the layers.
- **Z-index files** (e.g., `depths.ts`, `zIndex.ts`) are a bonus category — extract if present.

---

## Step 2 — Extract Design Tokens

For each detected category, extract raw values. Use subagents in parallel when auditing multiple categories or platforms.

### Categories to extract

#### Colors
- Count total definitions and unique values
- Map the architecture: how many layers? (e.g., core palette → semantic aliases → component overrides)
- Note naming conventions
- Identify dark mode / high contrast support
- Flag duplicates, near-duplicates (< 5% color distance), and orphaned legacy values
- **Computed colors:** If the codebase uses color manipulation functions (`darken()`, `lighten()`, `transparentize()`, `mix()`, etc.), report them as expressions, not resolved hex. Group them separately as "computed colors" so the reader knows these are dynamic.

#### Typography
- Font families (system vs custom)
- Type scale (sizes, weights, line heights, letter spacing)
- Responsive behavior (Dynamic Type, rem/em scaling, media queries)
- Naming convention (semantic vs raw)
- **Abstraction components:** Look for `Text`, `Heading`, `Label` wrapper components that enforce a subset of the type scale. These are the "intended" API even if many components bypass them with raw values. Report both the intended scale AND the ad-hoc usage, with a count of how many components bypass the abstraction.
- **Dual scales:** Some projects have different heading sizes for different contexts (e.g., page headings vs editor headings). If found, present them side-by-side and note whether the divergence is intentional or accidental.
- **i18n line-height overrides:** Check for script-specific line-height adjustments (CJK, Arabic, Devanagari, Thai, etc.). This is a sign of production-grade internationalization.

#### Spacing
- Is there a centralized scale? (4pt grid, 8pt grid, custom)
- Or are values scattered per-component?
- Responsive variants (breakpoints, orientation, screen-width qualifiers)
- Count distinct values and assess consistency

#### Corner Radius
- List all distinct values
- Are they semantically named or raw numbers?
- Component-specific vs shared

#### Elevation / Shadow
- Number of levels
- Naming convention
- Dark mode variants
- **Separate themed vs ad-hoc.** Report which shadows are in the theme system (and thus get light/dark variants) vs which are hardcoded in components (and thus break in dark mode).

#### Z-Index / Depth
- Look for a centralized z-index scale file (e.g., `depths.ts`, `zIndex.ts`, `z-index.scss`)
- If found, extract the full scale with names and values
- If not found, note that as a gap — competing z-index values cause layering bugs

#### Icons
- System (SF Symbols, Material Icons) vs custom vs library (react-icons, FontAwesome, Lucide, etc.)
- Icon font vs individual SVGs vs sprite sheet vs React components (npm package)
- Count and categorization
- **Registry pattern:** Check for a central icon registry/library that maps string keys to components (common in larger apps). Note whether icons are discoverable (keyword search) or just a flat list.
- **Hybrid systems:** Many projects combine 2-3 icon sources (e.g., custom SVGs + FontAwesome + emoji). Map which source covers which use case.

### Output format per category

For each category, produce a table in the audit report:

```markdown
| File | Purpose | Count | Naming Convention |
|------|---------|-------|-------------------|
```

---

## Step 3 — Assess Maturity

Rate each category on a 3-layer scale:

| Layer | Description | What it means |
|-------|-------------|---------------|
| **Layer 1 — Extraction** | Raw values exist but are scattered | "We found the tokens but they're not organized" |
| **Layer 2 — Patterns** | Consistent naming, centralized files, clear architecture | "There's a system here" |
| **Layer 3 — Recommendations** | Actionable improvements identified | "Here's what to fix and why" |

Produce an **Executive Summary** table:

```markdown
| Category | Found | Assessment |
|----------|-------|------------|
| **Colors** | 200+ across 10 files | Two coexisting systems. Well-organized but duplicated. |
| **Typography** | 9 styles + 5 custom fonts | Strong accessibility support. |
| **Spacing** | 100+ scattered constants | **No centralized scale.** Needs formalization. |
```

Then produce a **Strengths / Weaknesses** section. This is the most valuable part of the audit — it tells the reader what to protect and what to fix.

**Strengths** = things the codebase does well that should be preserved (mature color architecture, comprehensive z-index scale, i18n line-heights, centralized icon registry, etc.)

**Weaknesses** = things that are scattered, inconsistent, or missing that hurt maintainability.

Finally, produce a **Maturity Scorecard** table:

```markdown
| Category | Layer 1 (Extracted) | Layer 2 (Patterned) | Layer 3 (Actionable) |
|----------|:---:|:---:|:---:|
| **Colors** | Y | Y | -- (already mature) |
| **Typography** | Y | Partial | Y |
| **Spacing** | Y | N | Y |
| **Corner Radius** | Y | N | Y |
| **Elevation/Shadow** | Y | Partial | Y |
| **Z-Index** | Y | Y | -- (already mature) |
| **Icons** | Y | Y | -- (already mature) |
| **Breakpoints** | Y | Y | -- (already mature) |
```

Categories that score "Y" on Layer 2 should NOT get Layer 3 recommendations — they're already working. Only recommend changes for categories that are Partial or N on Layer 2.

---

## Step 4 — Component Deep-Dive (if requested)

When the user asks to audit a specific component (e.g., Button), follow this 3-phase pipeline: **Audit → HTML → JSON**. The HTML reference is the visual proof that catches CSS-to-Figma translation issues before they hit the plugin.

### Phase 1: Extract component spec from code

Read the component source files. Extract:
- Variant axes (style, size, state) and all values
- Props interface
- Per-variant visual properties: fill, text color, border, shadow, padding, font size/weight, corner radius, height, opacity
- Computed values: resolve `darken()`, `lighten()`, `transparentize()` to exact hex by installing the library from the project's `package.json` and running it (see Step 1 CSS-in-JS specifics). Never approximate HSL math manually.
- Note which properties come from the theme vs are hardcoded
- **Interaction states** — go beyond Enabled/Disabled. Search for:
  - **Android/Compose:** `InteractionSource`, ripple config, `ButtonDefaults.buttonColors()`, M3 state layer specs (hover 8%, focus 12%, pressed 12% of content color on container)
  - **iOS/UIKit:** `isHighlighted`, `isEnabled`, `dimsWhenHighlighted`, `dimsWhenDisabled`, `updateAlpha()`, background image swaps (`.normal` vs `.highlighted`), scale animations
  - **iOS/SwiftUI:** `buttonStyle`, `.opacity()` modifiers, `@Environment(\.isEnabled)`
  - **Web:** `:hover`, `:focus-visible`, `:active`, `outline`, box-shadow changes, background-color transitions
  - Document the *mechanism* (e.g., "M3 state layer overlay" or "whole-button opacity dim") not just the values — this tells the designer how to implement it in Figma

### Phase 2: Build HTML reference page

Create `<Component>.html` in the output directory that renders **every variant** using the actual code values. This is the visual source of truth.

Structure:
```html
<!-- One section per style variant -->
<h2>Primary</h2>
<div class="grid">
  <!-- One button per size × state -->
  <button class="btn btn-primary btn-large">Button</button>
  <button class="btn btn-primary btn-large" disabled>Button</button>
  ...
</div>
```

Rules for the HTML:
- **Use the exact CSS values from the code** — same font-weight, padding, border-radius, colors, shadows
- **Render all states.** Don't just show Enabled/Disabled. Show every interaction state the platform supports (Hovered, Focused, Pressed, etc.) as static buttons using CSS classes (e.g., `.is-hovered`, `.is-pressed`). For M3 state layers, use an inner `<span class="state-layer">` with absolute positioning and the correct rgba overlay.
- **Include dark mode** if the codebase supports it (render in a dark container)
- **Include the Figma capture script** in `<head>` so it can be imported to Figma later:
  `<script src="https://mcp.figma.com/mcp/html-to-design/capture.js" async></script>`
- Serve locally (`python -m http.server <port>`) and open in browser for visual verification
- **Add comprehensive reference tables** at the bottom (see below)

**The HTML is the checkpoint.** Visually verify it matches the running app before proceeding to JSON. If it looks wrong here, fix it before generating JSON.

### Phase 2b: Reference tables (bottom of HTML)

The reference tables are the designer-facing deliverable — a complete token manifest that documents every value needed to build the component in Figma, without requiring the designer to read JSON or source code. Include **all** of the following tables that apply:

#### Required tables

1. **Size Geometry** — one row per size variant:
   - Columns: Size | Padding V | Padding H | Font Size | Line Height | Font Weight | Radius | Computed Height
   - Note any size exceptions (e.g., a style that overrides padding at a specific size)

2. **Color Tokens — Enabled State** — one row per style (or style × size if colors differ by size):
   - Columns: Style | [swatch] | Fill | [swatch] | Text | Stroke | Stroke Weight
   - Use inline color swatches (`<span>` with background color + 1px border) for visual scanning
   - Use "transparent" (not blank) for transparent fills; use dashed-border swatch for visual clarity

3. **Interaction State Tokens** — how each state modifies the enabled appearance:
   - **For M3/state-layer systems (Android, Material):** Table with columns: Style | State Layer Color | Hovered (8%) | Focused (12%) | Pressed (12%) — show the rgba values
   - **For opacity-based systems (iOS, simple web):** Table with columns: State | Mechanism | Value | Source | Applies To
   - Document the *rule*, not just the value: "8% of content color" is more useful than just "rgba(255,255,255,0.08)" because it tells the designer how to derive new values

4. **Disabled State Tokens** — one row per style:
   - Columns: Style | [swatch] | Fill (Disabled) | [swatch] | Text (Disabled) | Stroke (Disabled)
   - Separate from enabled colors because disabled often uses a completely different color system (e.g., M3 uses onSurface at fixed alpha, not a dimmed version of the enabled color)

#### Conditional tables (include when applicable)

5. **Dark Mode Color Tokens** — if the codebase has dark mode:
   - Columns: Token Name | [swatch] | Light | [swatch] | Dark | Figma Variable Name
   - Include suggested Figma variable names (e.g., `colors/primary`, `colors/on-primary`)
   - Flag tokens that are static across modes (no dark variant)

6. **High Contrast Tokens** — if the codebase has accessibility color modes:
   - Columns: Token | Light | Dark | Light HC | Dark HC

7. **Actual vs Extrapolated Configurations** — if the component doesn't have a clean variant matrix:
   - Columns: Configuration | Style | Size | In Code? | Notes
   - This tells the designer which variants are real implementations vs gap-fills for a complete Figma component set

#### Always include: Figma Import Cheat Sheet

8. **Variant Property Axes** — the cartesian product definition:
   - Columns: Platform | Property | Values | Total Variants

9. **Figma Auto-Layout Settings** — base frame properties:
   - Columns: Property | Value (per platform if cross-platform)
   - Include: Layout Mode, Axis Alignment, Sizing Mode, Clips Content, Font Family, Font Weight

10. **Figma Variables Needed** — the complete variable list for the component:
    - Columns: Variable Name | Type (COLOR/FLOAT) | Light Value | Dark Value | Used By
    - Include both color variables and numeric variables (state layer opacities, specific radii, etc.)
    - This table is the single source of truth for what a designer needs to create in Figma Variables before building the component

### Phase 3: Generate JSON from HTML

Now produce the plugin-compatible JSON, using the **rendered HTML as the reference** for all values. This is where CSS-to-Figma translation happens:

- **Heights:** Inspect the rendered button heights in the HTML. Compute Figma padding using: `paddingV = (renderedHeight - textLineHeight) / 2`. Use a natural text lineHeight (20px for 14px font, 18px for 13px font), never the CSS line-height value.
- **Shadows:** Note which variants have shadows in the HTML. Record them in `_metadata` since the plugin doesn't support Figma `effects` yet. The HTML-to-Figma capture path preserves shadows automatically.
- **Computed colors:** Use the resolved hex values visible in the HTML (browser DevTools), not the code expressions.
- **Border tricks:** If CSS uses `box-shadow: inset` for borders, map to `strokeColor` + `strokeWeight` in the JSON.

### Component spec JSON schema

```json
{
  "name": "<project>-<platform>-<component>",
  "source": "<project>-<platform>-<language>",
  "sourceFiles": ["path/to/file1", "path/to/file2"],
  "renderType": "declarative",
  "variantProperties": {
    "Style": ["Primary", "Secondary", "..."],
    "Size": ["Large", "Medium", "Small"],
    "State": ["Enabled", "Hovered", "Focused", "Pressed", "Disabled"]
  },
  "componentProperties": {
    "label": { "type": "TEXT", "defaultValue": "Button" }
  },
  "base": {
    "layoutMode": "HORIZONTAL",
    "primaryAxisAlignItems": "CENTER",
    "counterAxisAlignItems": "AUTO",
    "primaryAxisSizingMode": "AUTO",
    "counterAxisSizingMode": "AUTO",
    "cornerRadius": 8,
    "fontFamily": "Inter",
    "fontWeight": 600
  },
  "children": [
    {
      "type": "TEXT",
      "name": "label",
      "propertyRef": { "characters": "label" }
    }
  ],
  "variants": [
    {
      "name": "Style=Primary, Size=Large, State=Enabled",
      "fillColor": { "fallback": "#2267F5" },
      "textColor": { "fallback": "#ffffff" },
      "paddingLeft": 16, "paddingRight": 16,
      "paddingTop": 10, "paddingBottom": 10,
      "fontSize": 14, "lineHeight": 20
    }
  ],
  "_metadata": {
    "actualConfigurations": ["list of variants that exist in code"],
    "missingCombinations": ["variant combos not implemented"],
    "unsupportedByPlugin": ["box-shadow: rgba(0,0,0,0.2) 0px 1px 2px — use HTML capture for shadow fidelity"],
    "notes": "any platform-specific caveats"
  }
}
```

### Dual import paths

The HTML and JSON serve different purposes in Figma:
- **JSON → plugin import:** Creates a proper Figma component set with variant properties. Best for structured design system work. Missing: shadows, complex borders.
- **HTML → Figma capture:** Creates a flat visual reference with pixel-perfect fidelity (shadows, gradients, everything the browser renders). Best for visual comparison and drift detection.

Use both: import the JSON as the working component set, capture the HTML as a side-by-side reference to verify accuracy.

### Plugin compatibility constraints

The component spec JSON is consumed by the Code Bridge Figma plugin (`bootstrap-figma-to-code-and-back-again/plugin/src/code.ts`). The following rules MUST be followed or the import will crash:

1. **`fillColor` and `textColor` must always be objects with a `fallback` key.** The plugin calls `hex.replace('#', '')` on `ref.fallback`. If `fallback` is missing or the value is a bare string (e.g., `"inherit"`, `"none"`), the plugin crashes with `TypeError: Cannot read property 'replace' of undefined`.
   - Transparent: use `{ "fallback": "#FFFFFF00" }` (hex with alpha)
   - Inherited/none: use `{ "fallback": "#00000000" }` (fully transparent)
   - Computed values: pre-resolve to static hex. Store the expression in `_metadata.notes` for documentation.

2. **All dimension values must be numbers, not strings.** `height`, `fontSize`, `lineHeight`, `paddingLeft`, etc. must be numeric. Never use `"auto"`, `"inherit"`, or string values.

3. **`children` must be flat Figma-compatible nodes** — `TEXT`, `FRAME`, or `COMPONENT_INSTANCE`. Do not use `"visible": "conditional"` or other non-Figma properties. The plugin's `buildDeclChildren` function only handles `FRAME`, `TEXT`, and `COMPONENT_INSTANCE` types.

4. **Only include properties the plugin processes:** `fillColor`, `textColor`, `strokeColor`, `strokeWeight`, `paddingTop/Bottom/Left/Right`, `fontSize`, `lineHeight`, `letterSpacing`, `opacity`, `cornerRadius`, `textCase`, `effects`. Properties like `boxShadow` (CSS string), `border`, `width` (on variants) are ignored.

5. **Capture shadows via `effects` in Figma-native format.** The plugin reads `effects` arrays on each variant and applies them as Figma effects (DROP_SHADOW, INNER_SHADOW) in the Effects panel. Use Figma's `EffectType.DROP_SHADOW` schema:
   ```json
   "effects": [{ "type": "DROP_SHADOW", "color": { "r": 0, "g": 0, "b": 0, "a": 0.2 }, "offset": { "x": 0, "y": 1 }, "radius": 2, "spread": 0, "visible": true }]
   ```
   - Map CSS `box-shadow: rgba(R,G,B,A) Xpx Ypx Rpx` → `{ color: {r,g,b,a}, offset: {x,y}, radius: R }`
   - CSS `box-shadow: inset ...` → use `strokeColor`/`strokeWeight` instead (Figma has no inset shadow)
   - Disabled variants with no shadow: use `"effects": []`
   - Both import paths now support shadows: JSON→plugin creates Figma effects directly, HTML→Figma capture renders them visually.

6. **Exclude non-renderable variants.** Components with fundamentally different DOM structures (like ButtonLink, NudeButton) should go in `_metadata.excludedFromSpec`, not in the `variants` array. Interaction states (hover, focus) ARE renderable and SHOULD be included as static variants.

6. **Height = padding + lineHeight in Figma.** CSS and Figma handle height differently:
   - In CSS, `line-height: 32px` on a flex child creates a 32px box, and `padding: 0` adds nothing — total height 32px.
   - In Figma auto-layout (HUG), the text node's lineHeight is its content height, and frame padding is added ON TOP — so `lineHeight: 32` + `padding: 0` = 32px, but `lineHeight: 32` + `padding: 4` = 40px (not 36px as CSS would give).
   - **Fix:** Use a natural text lineHeight (e.g., 20px for 14px font, 18px for 13px font) and redistribute the remaining space to padding.
   - Formula: `paddingTop = paddingBottom = (targetHeight - textLineHeight) / 2`
   - Example: 40px button with 14px font → lineHeight: 20, paddingTop: 10, paddingBottom: 10
   - Always set `counterAxisSizingMode: "AUTO"` (hug), never `"FIXED"`.

7. **`primaryAxisSizingMode` and `counterAxisSizingMode` only accept `"FIXED"` or `"AUTO"`.** Never use `"FILL"` — the Figma Plugin API will throw `Invalid enum value`. Fill-width/fill-height behavior is set via `layoutSizingHorizontal: "FILL"` or `layoutSizingVertical: "FILL"` on the node *after* it's placed inside a parent's auto-layout. In component spec JSON, always use `"AUTO"` for the base and let the designer set fill at the instance level.

8. **Variant names must exactly match** the cartesian product of `variantProperties` keys/values, formatted as `Key=Value, Key=Value`. The plugin generates names via `vpKeys.map(k => k + '=' + combo[k]).join(', ')` and matches them against `variants[].name`.

---

## Step 5 — Cross-Platform Comparison (if multiple platforms)

When auditing the same product across platforms, produce a comparison report.

### Structure

1. **At a Glance** — side-by-side summary table (files, language, framework, counts per category)
2. **Per-category comparison** — for each token category:
   - Architecture differences
   - Value differences (e.g., primary blue: iOS `#2267F5` vs Android `#2C58C3`)
   - Feature gaps (e.g., iOS has high contrast, Android doesn't)
3. **Critical Inconsistencies** — numbered list of divergences that would confuse users or designers
4. **Recommendations** — which platform's approach to adopt for each category

---

## Step 6 — Code vs Figma Comparison (if Figma URL provided)

When the user provides a Figma file or component URL:

1. Use `get_design_context` or `get_metadata` to extract Figma component properties
2. Compare against the code-extracted component spec JSON
3. Produce a drift report:

```markdown
| Property | Figma | Code | Match? |
|----------|-------|------|--------|
| Corner radius | 8px | 12px | ❌ Drift |
| Primary fill | #2267F5 | #2267F5 | ✅ |
| Font weight | 600 | 500 | ❌ Drift |
```

For automated diff, use the `compare.mjs` pattern from the Bootstrap/MUI bridge projects if a compare script exists in the project.

---

## Step 7 — Write Outputs

Save all artifacts to the output directory:

| File | Content |
|------|---------|
| `<platform>-design-system-audit.md` | Full audit report (Executive Summary → Strengths/Weaknesses → Layer 1 extraction → Layer 2 patterns → Layer 3 recommendations → Maturity Scorecard) |
| `<component>.html` | HTML reference page rendering all variants with actual code values (Phase 2 — visual proof) |
| `<Component>-<platform>.json` | Component spec JSON derived from HTML reference (Phase 3 — Figma plugin import) |
| `cross-platform-comparison.md` | Side-by-side comparison (only if multiple platforms) |
| `drift-report.md` | Figma vs code drift (only if Figma URL provided) |

### Audit report structure (in order)

1. **Header** — project name, date, source path
2. **Executive Summary** — table with category / found / assessment
3. **Stack** — framework, styling approach, build tool, dark mode mechanism, breakpoints
4. **Strengths / Weaknesses** — what to protect and what to fix
5. **Layer 1: Raw Token Extraction** — per-category tables with file, purpose, count, naming convention
6. **Layer 2: Pattern Assessment** — narrative analysis of architecture, naming, centralization
7. **Layer 3: Recommendations** — each recommendation cites specific Layer 1/2 findings as evidence
8. **Maturity Scorecard** — final Y/Partial/N grid

---

## Autonomy Rules

**Run automatically (no confirmation needed):**
- Reading files, globbing, grepping within the target codebase
- Creating audit output files in the output directory
- Shallow-cloning a repo the user provided

**Ask before:**
- Cloning repos not explicitly provided by the user
- Making any changes to the audited codebase itself
- Accessing Figma files the user hasn't linked

---

## Ground Rules

- **Extract, don't guess.** Every value in the audit must trace back to a file and line number. If you can't find a value, say so — don't infer from naming patterns.
- **Adapt to the stack.** The categories and file patterns above are starting points. If the codebase uses an unusual token system (e.g., Tailwind plugin, custom build step, generated tokens), follow the actual architecture.
- **Parallelize extraction.** Use subagents to audit multiple categories or platforms concurrently.
- **Keep reports scannable.** Lead with the Executive Summary table. Put raw extraction details in collapsed sections or appendices if the report gets long.
- **Component specs are Figma-compatible.** The JSON schema uses Figma property names (`fillColor`, `paddingLeft`, `layoutMode`) so specs can be directly compared against Figma extractions or imported into the plugin pipeline.
- **No recommendations without evidence.** Layer 3 recommendations must reference specific findings from Layer 1/2. "Consider formalizing spacing" is only valid if you found scattered, inconsistent spacing values.
- **Report token adoption %.** For each category, estimate what percentage of usages reference the centralized tokens vs hardcoded values. Example: "Spacing: ~40% token-based, ~60% hardcoded." This gives the reader a clear migration target.
- **Handle computed values.** Color manipulation functions (`darken()`, `lighten()`, `transparentize()`, `mix()`, `rgba()` with variable base) should be reported as expressions, not resolved hex. Group them as "computed" in the color tables.
- **Dark mode completeness.** For CSS-in-JS themes, check whether ad-hoc hardcoded colors in components get light/dark variants. Shadows and colors outside the theme system are the most common dark-mode bugs — flag them explicitly.
- **Read the project's existing CLAUDE.md or AGENTS.md** if present — it may reveal architecture decisions that affect how tokens are organized.
