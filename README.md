# Code Audit Skill for Claude Code

Extract and audit design systems from any codebase. Auto-discovers every UI component, generates an interactive inventory, then produces designer-facing HTML reference pages and Figma-importable JSON specs.

## What It Does

Point it at a codebase. It will:

1. **Auto-discover every UI component** — scans the repo, classifies components as Atom / Molecule / Organism / Template, and generates an interactive `inventory.html` with live filtering
2. **Ask what to audit** — you pick one, several (comma-separated), or batch by tier ("all atoms")
3. **Deep-dive each component** — produces per-component:
   - **HTML reference page** with every variant rendered statically (all sizes, styles, interaction states, dark mode)
   - **JSON component spec** in Figma-native format (importable via the Code Bridge plugin)
   - **Reference tables** documenting every token a designer needs to build the component in Figma
   - **Figma Import Cheat Sheet** — auto-layout settings, variables needed, variant property axes
4. **Extract design tokens** — colors, typography, spacing, radius, elevation, z-index, icons
5. **Cross-platform comparison** — side-by-side iOS vs Android token diff
6. **Figma drift detection** — compare code tokens against a Figma file

## Tested On

| Repo | Framework | Components Found | Discovery Unit |
|------|-----------|-----------------|----------------|
| Outline | React + styled-components | 199 | Individual `.tsx` files |
| shadcn/ui | React + Tailwind v4 / CVA | 56 | Individual `.tsx` files |
| Angular Material | Angular 22 + Sass M3 | 35 modules | Module directories |
| Signal Android | Kotlin + Compose M3 | 75 button variants | `@Composable` functions |
| Signal iOS | Swift + UIKit | 27 button variants | `UIView` subclasses |

## Install

Copy the `code-audit` folder into your Claude Code skills directory:

```
~/.claude/skills/code-audit/SKILL.md
```

On Windows:
```
C:\Users\<you>\.claude\skills\code-audit\SKILL.md
```

That's it. Claude Code will auto-discover the skill.

## Usage

```
/code-audit ~/projects/my-app
```

Or just ask naturally:

- "audit the design system in this repo"
- "extract design tokens from the codebase"
- "compare the iOS and Android design systems"

The skill auto-detects the stack, discovers all components, shows you the inventory, and asks what to audit. No need to specify a component name upfront.

### Batch Mode

After the inventory is displayed, you can pick multiple components:

```
> Badge, Button, Checkbox
> all atoms
> all molecules
```

Each component runs in parallel via subagents.

## What You Get

### inventory.html (always first)

Interactive component inventory with:
- Summary strip (total count per tier)
- Stack tags (framework, styling, build tool)
- Filterable tables by tier and text search
- Line-count bars for visual complexity scanning
- Figma capture script for import

### Component HTML Reference Pages

Every variant rendered with exact code values — sizes, styles, all interaction states (hover, focus, pressed, disabled), dark mode. Includes:

- Visual variant grid
- Size geometry table
- Color tokens with inline swatches
- Interaction state tokens (M3 state layers, opacity dims, CSS hover rules)
- Disabled state tokens
- Dark mode color mapping
- **Figma Import Cheat Sheet** — auto-layout settings, variables needed (with light/dark values), variant property axes

### JSON Component Spec

Figma-native format with:
- Variant properties (Style x Size x State)
- Per-variant: fillColor, textColor, strokeColor, strokeWeight, padding, fontSize, lineHeight, cornerRadius, opacity
- `effects` array with DROP_SHADOW/INNER_SHADOW in Figma's native schema
- `_metadata` with excluded components, state layer documentation, and architecture notes
- Separate specs for components with different geometries (e.g., standard buttons vs FABs vs icon buttons)

Import via the **Code Bridge Figma Plugin** (separate package) to create a full Figma component set with variant properties and effects.

## Supported Stacks

| Stack | Token Sources | Discovery Strategy |
|-------|--------------|-------------------|
| Web (React/Vue/Svelte) | JSX exports, theme objects | Individual `.tsx`/`.vue`/`.svelte` files |
| Web (Angular) | `@Component` decorators, SCSS themes | Module directories (one per component) |
| Web (CSS/SCSS) | Custom properties, SCSS variables | File patterns |
| Web (CSS-in-JS) | styled-components, Emotion + polished/color2k | Theme objects + TypeScript types |
| Web (Tailwind) | Tailwind config, CVA variants, CSS custom properties | Utility classes + config |
| iOS (UIKit) | UIColor extensions, Asset catalogs | `UIView`/`UIControl` subclasses |
| iOS (SwiftUI) | Color extensions, Asset catalogs | `struct ... : View` |
| Android (XML) | colors.xml, dimens.xml, styles.xml, themes.xml | Layout XML + custom views |
| Android (Compose) | Theme.kt, Color.kt, Type.kt | `@Composable fun` |
| React Native | StyleSheet objects, theme providers | Component files |
| Flutter | ThemeData, ColorScheme | `StatelessWidget`/`StatefulWidget` |

## Key Features

- **Auto-discovery** — no need to name components upfront; the skill scans and presents an inventory
- **Batch auditing** — pick multiple components; each runs in parallel
- **Verified computed colors** — installs polished/color2k from the project's package.json and runs it to get exact hex values (no manual HSL approximation)
- **All interaction states** — not just Enabled/Disabled. Extracts hover, focus, pressed, and documents the mechanism (M3 state layers, opacity dims, CSS pseudo-classes)
- **Figma Effects panel** — shadows captured in Figma-native DROP_SHADOW format, imported directly to the Effects panel
- **Figma Import Cheat Sheet** — every HTML page includes auto-layout settings, complete variable list, and variant property axes
- **Dark mode** — full light/dark token mapping with suggested Figma variable names
- **Cross-platform** — same skill handles web (React, Angular, Vue), iOS, and Android with platform-specific extraction rules
- **Geometry-aware JSON** — automatically splits components with different frame structures into separate Figma component sets

## Requirements

- [Claude Code](https://claude.com/claude-code) CLI
- For Figma import: Code Bridge Figma Plugin (separate package)
