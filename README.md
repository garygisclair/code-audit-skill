# Code Audit Skill for Claude Code

Extract and audit design systems from any codebase. Produces designer-facing HTML reference pages and Figma-importable JSON specs.

## What It Does

Point it at a codebase and a component name. It will:

1. **Detect the tech stack** — web (CSS/SCSS/Tailwind/CSS-in-JS), iOS (UIKit/SwiftUI), Android (XML/Compose), React Native, Flutter
2. **Extract every design token** — colors, typography, spacing, radius, elevation, z-index, icons
3. **Deep-dive a specific component** — produces:
   - **HTML reference page** with every variant rendered statically (all sizes, styles, interaction states, dark mode)
   - **JSON component spec** in Figma-native format (importable via the Code Bridge plugin)
   - **Reference tables** documenting every token a designer needs to build the component in Figma
4. **Cross-platform comparison** — side-by-side iOS vs Android token diff
5. **Figma drift detection** — compare code tokens against a Figma file

## Install

1. Copy the `code-audit` folder into your Claude Code skills directory:

```
~/.claude/skills/code-audit/SKILL.md
```

On Windows:
```
C:\Users\<you>\.claude\skills\code-audit\SKILL.md
```

2. That's it. Claude Code will auto-discover the skill.

## Usage

```
/code-audit ~/projects/my-app
/code-audit ~/projects/my-app --focus Button
/code-audit ~/projects/my-app --focus Input --output ~/projects/my-app/audit
```

Or just ask naturally:

- "audit the design system in this repo"
- "extract the Button component tokens from the codebase"
- "compare the iOS and Android design systems"

## What You Get

### HTML Reference Page

Every variant rendered with exact code values — sizes, styles, all interaction states (hover, focus, pressed, disabled), dark mode. Includes the Figma capture script for HTML-to-Figma import.

Bottom of the page has comprehensive reference tables:
- Size geometry (padding, font, radius, computed height)
- Color tokens with inline swatches
- Interaction state tokens (M3 state layers, opacity dims, CSS hover rules)
- Disabled state tokens
- Dark mode color mapping with suggested Figma variable names
- Figma import cheat sheet (variant axes, auto-layout settings, full variable list)

### JSON Component Spec

Figma-native format with:
- Variant properties (Style × Size × State)
- Per-variant: fillColor, textColor, strokeColor, strokeWeight, padding, fontSize, lineHeight, cornerRadius, opacity
- `effects` array with DROP_SHADOW/INNER_SHADOW in Figma's native schema
- `_source` section documenting theme tokens, computed colors, interaction mechanisms
- `_metadata` with excluded components and architecture notes

Import via the **Code Bridge Figma Plugin** (separate package) to create a full Figma component set with variant properties and effects.

## Supported Stacks

| Stack | Token Sources |
|-------|--------------|
| Web (CSS/SCSS) | Custom properties, SCSS variables, Tailwind config |
| Web (CSS-in-JS) | Theme objects, styled-components, Emotion + polished/color2k |
| iOS (UIKit) | UIColor extensions, Asset catalogs |
| iOS (SwiftUI) | Color extensions, Asset catalogs |
| Android (XML) | colors.xml, dimens.xml, styles.xml, themes.xml |
| Android (Compose) | Theme.kt, Color.kt, Type.kt |
| React Native | StyleSheet objects, theme providers |
| Flutter | ThemeData, ColorScheme |

## Key Features

- **Verified computed colors** — installs polished/color2k from the project's package.json and runs it to get exact hex values (no manual HSL approximation)
- **All interaction states** — not just Enabled/Disabled. Extracts hover, focus, pressed, and documents the mechanism (M3 state layers, opacity dims, CSS pseudo-classes)
- **Figma Effects panel** — shadows captured in Figma-native DROP_SHADOW format, imported directly to the Effects panel
- **Dark mode** — full light/dark token mapping with suggested Figma variable names
- **Cross-platform** — same skill handles web, iOS, and Android with platform-specific extraction rules

## Requirements

- [Claude Code](https://claude.com/claude-code) CLI
- For Figma import: Code Bridge Figma Plugin (separate package)
