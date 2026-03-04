# Synergy Design System — CSS Variables POC

> A proof-of-concept demonstrating how **Figma Variables** (Modes and Extended Collections) translate into **CSS custom properties** using the cascade.

**Live demo:** open `index.html` via a local HTTP server (see [Running Locally](#running-locally)).

---

## What This Demonstrates

Figma's Variables system has two distinct organization paradigms:

| Figma Concept | What It Is | CSS Equivalent |
|---|---|---|
| **Modes within a collection** | Light / Dark axes on the same collection (e.g., a "Semantics" collection with a Light mode and a Dark mode) | `data-mode` attribute on `<body>` — overridden by a higher-specificity descendant selector |
| **Extended Collections** | A separate collection that overrides a base (e.g., "Brand A" extends "Synergy") | `data-brand` attribute on `<html>` — separate selector at the same specificity level |

**The key conceptual gap:** CSS has no mechanism to distinguish between these two paradigms. Both become `data-*` attributes, and the hierarchy between them is reconstructed purely through **CSS selector specificity**. The four-layer architecture below is the translation model.

---

## Architecture

### The Four Layers

```
Layer 1 — Primitives      :root                                    ← raw palette values
Layer 2 — Brand           [data-brand="synergy"]                   ← brand expression
Layer 3a — Accent         [data-brand][data-accent="primary"]      ← semantic token set
Layer 3b — Mode           [data-brand][data-accent] [data-mode]    ← light/dark override
Layer 4 — Components      .preview-box, .btn, etc.                 ← component-local aliases
```

### Cascade Hierarchy (specificity low → high)

| Layer | Selector Example | Specificity | Purpose |
|---|---|---|---|
| Primitives | `:root` | `0-0-0` | Raw color values (`--color-blue-500: #3b82f6`) |
| Brand | `[data-brand="synergy"]` | `0-1-0` | Brand roles (`--brand-primary: var(--color-blue-500)`) |
| Accent | `[data-brand][data-accent="primary"]` | `0-2-0` | Semantic tokens (`--ui-elements-fill-loud: var(--brand-primary)`) |
| Mode | `[data-brand][data-accent] [data-mode="dark"]` | `0-3-0` | Dark overrides (`--backgrounds-default: var(--color-neutral-950)`) |
| Component | `.preview-box` | `0-1-0` | Local aliases (`--preview-bg: var(--backgrounds-default)`) |

### HTML State Structure

```html
<html lang="en" data-brand="synergy" data-accent="neutral">
  <body data-mode="light">
    <!-- components here -->
  </body>
</html>
```

`data-brand` and `data-accent` live on `<html>`. `data-mode` lives on `<body>`. This enables the mode selector to use the **descendant combinator** (space character), which achieves the necessary specificity:

```css
/* This selector matches body when html has both data-brand and data-accent */
[data-brand][data-accent] [data-mode="dark"] { /* specificity 0-3-0 */ }

/* This would FAIL — all three attrs cannot exist on the same element */
/* [data-brand][data-accent][data-mode] — would never match */
```

---

## Token Naming Convention

Figma uses forward slashes to organize variables into groups. CSS custom properties use dashes:

```
Figma variable path        →   CSS custom property
─────────────────────────────────────────────────────
Color/Blue/500             →   --color-blue-500
Backgrounds/Default        →   --backgrounds-default
UI Elements/Fill/Loud      →   --ui-elements-fill-loud
Foregrounds/Link           →   --foregrounds-link
```

---

## Semantic Token Reference

### Surface Tokens (set by accent, overridden by mode)

| Token | Light Value | Dark Value |
|---|---|---|
| `--backgrounds-default` | `#ffffff` | `#0a0a0a` |
| `--backgrounds-raised` | `#fafafa` | `#171717` |
| `--backgrounds-lowered` | `#f5f5f5` | `#262626` |
| `--foregrounds-primary` | `#171717` | `#fafafa` |
| `--foregrounds-secondary` | `#525252` | `#a3a3a3` |
| `--foregrounds-tertiary` | `#a3a3a3` | `#737373` |
| `--foregrounds-link` | `var(--brand-primary)` | `var(--brand-primary-subtle)` |
| `--border-default` | `#e5e5e5` | `#404040` |
| `--border-strong` | `#a3a3a3` | `#737373` |
| `--focus-ring` | `var(--brand-primary)` | `var(--brand-primary-subtle)` |

### UI Element Tokens (set by accent)

Each accent maps these three variants (loud / normal / quiet) × three properties (fill / on / border):

```
--ui-elements-fill-loud     --ui-elements-on-loud     --ui-elements-border-loud
--ui-elements-fill-normal   --ui-elements-on-normal   --ui-elements-border-normal
--ui-elements-fill-quiet    --ui-elements-on-quiet    --ui-elements-border-quiet
```

- **Loud** → solid filled button (highest prominence)
- **Normal** → outlined button (secondary prominence)
- **Quiet** → ghost button (lowest prominence)

---

## Brand Differences

| Role | Synergy | Brand A | Brand B |
|---|---|---|---|
| primary | blue `#3b82f6` | orange `#f97316` | purple `#a855f7` |
| secondary | neutral-800 | neutral-700 | neutral-800 |
| highlight | purple `#a855f7` | purple `#a855f7` | **orange** `#f97316` |
| positive | green `#22c55e` | green (same) | green (same) |
| negative | red `#ef4444` | red (same) | red (same) |
| informative | teal `#14b8a6` | teal (same) | teal (same) |
| attention | yellow `#eab308` | yellow (same) | yellow (same) |

Positive, negative, informative, and attention are **universal semantics** — they do not change between brands. Only primary, secondary, and highlight are brand-differentiating.

---

## The Conceptual Gap (and How We Bridge It)

In Figma:
- **Modes** exist *within* a single collection as parallel value sets (Light ↔ Dark). Changing the mode switches the entire collection's active values simultaneously.
- **Extended Collections** are *separate* collections that inherit from a base but override specific variables. A "Brand A" collection might extend "Synergy" and only redefine the primary color.

In CSS:
- **Both concepts flatten into `data-*` attributes.** There is no native CSS concept of "this attribute is a mode" vs. "this attribute is an extension."
- The distinction is reconstructed through **selector specificity** and **cascade order**: higher-specificity selectors override lower ones; later-loaded files at equal specificity override earlier ones.
- The **descendant combinator** trick (`[html-attrs] [body-attr]`) is the only way to achieve different specificity levels for attributes on different ancestor elements.

**Implication for tooling:** Any tool that exports Figma Variables to CSS must decide how to encode the Mode vs. Extended Collection distinction. The approach here (data attributes + selector specificity) is one valid strategy. Alternatives include `@layer` (CSS Cascade Layers) or CSS `color-scheme` property for light/dark specifically.

---

## File Structure

```
synergy-poc/
├── tokens/
│   ├── 01-primitives.css              # :root — raw palette
│   ├── 02-brand-themes/
│   │   ├── synergy.css                # [data-brand="synergy"]
│   │   ├── brand-a.css                # [data-brand="brand-a"]
│   │   └── brand-b.css                # [data-brand="brand-b"]
│   ├── 03-semantics/
│   │   ├── accents/
│   │   │   ├── neutral.css            # [data-brand][data-accent="neutral"]
│   │   │   ├── primary.css            # [data-brand][data-accent="primary"]
│   │   │   ├── secondary.css
│   │   │   ├── positive.css
│   │   │   ├── negative.css
│   │   │   ├── informative.css
│   │   │   ├── highlight.css
│   │   │   └── attention.css
│   │   └── modes/
│   │       ├── light.css              # [data-brand][data-accent] [data-mode="light"]
│   │       └── dark.css               # [data-brand][data-accent] [data-mode="dark"]
│   └── 04-components/
│       └── preview-box.css            # component-scoped aliases
├── index.html                         # interactive demo
└── README.md
```

---

## Running Locally

This is a zero-dependency, pure HTML/CSS/JS project. You need a local HTTP server because browsers restrict CSS file loading from `file://` URLs.

**Option 1 — Node.js (npx, no install):**
```sh
cd synergy-poc
npx serve .
# Open http://localhost:3000
```

**Option 2 — Python:**
```sh
cd synergy-poc
python3 -m http.server 8080
# Open http://localhost:8080
```

**Option 3 — VS Code Live Server:**
Open `synergy-poc/index.html` in VS Code and click "Go Live" in the status bar (requires [Live Server extension](https://marketplace.visualstudio.com/items?itemName=ritwickdey.LiveServer)).

---

## GitHub Pages Deployment

1. Push the repository to GitHub.
2. In the repository Settings → Pages:
   - **Source:** Deploy from a branch
   - **Branch:** `main` (or whichever branch contains this code)
   - **Folder:** `/synergy-poc` (if the POC is in a subdirectory) or `/` (if at root)
3. GitHub will serve the site at `https://{username}.github.io/{repo}/synergy-poc/`.

> If you move `index.html` and `tokens/` to the repository root (not inside `synergy-poc/`), set the Pages folder to `/` and the site serves from `https://{username}.github.io/{repo}/`.

---

## Extending the POC

**Add a new brand:**
1. Create `tokens/02-brand-themes/brand-c.css` with `[data-brand="brand-c"]` selectors.
2. Add a radio button in `index.html`.
3. No other files need to change — the accent and mode layers reference brand tokens generically.

**Add a new accent:**
1. Create `tokens/03-semantics/accents/my-accent.css` following the pattern of any existing accent file.
2. Add the `<link>` in `index.html` after the other accent files.
3. Add a radio button in `index.html`.

**Add a new mode:**
1. Create a mode file (e.g., `tokens/03-semantics/modes/high-contrast.css`) with the `[data-brand][data-accent] [data-mode="high-contrast"]` selector pattern.
2. Add the `<link>` after `dark.css`.
3. Add a radio button in `index.html`.
