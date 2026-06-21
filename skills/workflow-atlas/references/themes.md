# Themes & color reference

Authoritative: `SPEC.md` §5, §10. This is the working guide.

## The 9 bundled presets

| `--theme` / `data-theme` | Mode  | Family    | Use                                            |
|--------------------------|-------|-----------|------------------------------------------------|
| `okabe-ito-light`        | light | Okabe-Ito | **Default.** Figures, legends, color-vision safe |
| `okabe-ito-dark`         | dark  | Okabe-Ito | Same viz palette on dark chrome                 |
| `catppuccin-latte`       | light | Catppuccin| UI-cohesive light                              |
| `catppuccin-mocha`       | dark  | Catppuccin| UI-cohesive dark                               |
| `solarized-light`        | light | Solarized | Document/JSON/log adjacency                     |
| `solarized-dark`         | dark  | Solarized | Classic dark                                   |
| `nord`                   | dark  | Terminal  | Calm blue dark                                 |
| `dracula`                | dark  | Terminal  | High-vibe dark (matches the original prototype) |
| `gruvbox-dark`           | dark  | Terminal  | Warm retro dark                                |

Each lives as `assets/themes/<id>.json` (and is embedded in `template.html`). The built HTML
carries **all nine** and a header dropdown to switch live — so you build with one default but the
reader can flip through them.

## Why `okabe-ito-light` is the default

Per the color research: terminal themes (Monokai/Dracula/Nord…) are tuned for *editor vibe*, not
*figure legibility*. For an artifact whose whole purpose is reading, **visualization palettes win**
— Okabe-Ito is a color-vision-deficiency-safe qualitative palette, on a light chrome that sits well
next to docs, JSON, and logs. Use a dark terminal theme for presentation/mood, not for dense study.

## How a node gets its color (hybrid, per channel)

For each group, the three node colors `(stroke, fill, sub)` resolve as:

1. **Explicit override** — if the group sets `stroke`/`fill`/`sub` in the data, that wins.
2. **Theme assignment** — otherwise `accent = theme.lanes[groupIndex % lanes.length]`, and the
   missing channels derive: `fill = mix(accent, surface, 0.86)`, `sub = mix(accent, text, 0.45)`.

So:

```jsonc
{ "id": "enrich", "label": "Enrich" }                  // fully theme-driven (recommended)
{ "id": "enrich", "label": "Enrich", "stroke": "#5bc8a0" }  // fixed border, fill/sub derived
{ "id": "brand",  "label": "Brand", "stroke": "#E4007F", "fill": "#2a0f1d", "sub": "#f0a" } // all fixed
```

**Recommendation:** leave groups color-free so theme switching recolors everything. Override only a
lane that must be a fixed brand color. Group **order** controls which palette slot each lane gets —
reorder `groups[]` to recolor without touching node data.

## Theme file shape

```jsonc
{
  "id": "okabe-ito-light",
  "name": "Okabe-Ito (Light)",
  "mode": "light",                 // affects derivation surface (light vs dark)
  "chrome": {                      // → CSS custom properties on :root
    "bg": "#F8FAFC", "surface": "#FFFFFF", "panel": "#F1F5F9",
    "line": "#CBD5E1", "grid": "#E2E8F0", "text": "#111827", "muted": "#64748B",
    "edge": "#94A3B8", "edge-active": "#0072B2", "edge-active-glow": "rgba(0,114,178,0.35)"
  },
  "lanes": ["#0072B2","#009E73","#E69F00","#CC79A7","#56B4E9","#D55E00","#000000"]
}
```

- `chrome` paints the document shell + the active-flow accent. The engine reads everything through
  these CSS custom properties; no color is hardcoded in script.
- `lanes` is the ordered **categorical** palette assigned to groups.

## Adding a custom theme

1. Add an entry to the `#atlas-themes` table in `assets/template.html` (and, optionally, a matching
   `assets/themes/<id>.json` for reuse). Keep the same shape.
2. Build with `--theme <id>`, or pick it from the dropdown.

## Accessibility rules (do not skip)

- Node-title text on its fill **must** meet WCAG AA (≥ 4.5:1). All bundled presets do.
- Category is conveyed by color **and** column position **and** label — never color alone.
- `lanes` palettes are **qualitative only**. Do not use them for scores/heat/probability — that is
  a deferred v2 feature using sequential/diverging palettes (ColorBrewer / Paul Tol).
- Don't put pale colors (e.g. Okabe-Ito yellow `#F0E442`) on thin lines or small text.
