# Workflow Atlas — Specification

**Status:** Draft v1 (2026-06-21)
**Skill name:** `workflow-atlas`
**Audience:** implementers of the skill, authors of atlas data files, and reviewers.

> A *Workflow Atlas* is not a drawing. It is a **specification you read interactively**:
> a project's **state nodes** and the **operations (flows)** that transform between them,
> declared as JSON and rendered as a single self-contained HTML document.

---

## Normative language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHOULD**, **SHOULD NOT**,
**RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described
in RFC 2119. In this spec they constrain three distinct artifacts; each requirement names its
target explicitly:

- **the Engine** — the JavaScript in `template.html` that renders and drives the atlas.
- **the Builder** — the `atlas build` script that injects data + theme into the template.
- **an Atlas** (or "the data") — a single JSON document conforming to §4.

---

## 0. Problem statement

Architecture diagrams drawn by hand (or by mermaid/graphviz) answer "what connects to what"
but decay the moment the system changes, and they cannot answer "what operation moves the system
from state A to state B, and in what order do the steps run." Teams instead want to **read** a
system: pick an operation, walk its steps, see exactly which states it touches and what payload
passes between them — without leaving a single HTML file that renders offline with no dependencies.

Workflow Atlas specifies that artifact. The author maintains **only a JSON document**; a default
HTML template carries a generic rendering engine; a build step (or an inline edit) fuses the
two into a portable spec. The reference engine is `skills/workflow-atlas/assets/template.html`.

---

## 1. Goals & non-goals

### 1.1 Goals

1. **JSON-first authoring.** Producing or updating an atlas **SHOULD** require editing only the
   data document — never the engine, CSS, or theme code (§8).
2. **Single self-contained artifact.** A built atlas **MUST** be one HTML file that renders fully
   offline from `file://` with zero network requests and zero external assets.
3. **State × operation model.** The data model **MUST** distinguish *nodes* (states/components)
   from *flows* (operations that transform between them), each flow being an ordered list of steps
   (§3).
4. **Readable, not just pretty.** Theming **MUST** prioritize the information-separation criteria
   from the color research (contrast, category/state separation, no meaning-by-color-alone) over
   aesthetic mimicry of terminal themes (§5, §10).
5. **Swappable themes.** Re-theming a built atlas **MUST** be possible by changing one selector
   (`--theme`) without touching the data (§5).
6. **Keyboard-walkable.** A reader **MUST** be able to advance and rewind through an operation's
   steps with the arrow keys, and select operations with number keys (§7).

### 1.2 Non-goals (v1)

1. **No per-node status layer.** v1 colors nodes by *lane (category)* only. A separate state-color
   layer (success / warning / error / running / disabled) is deferred to v2 (§5.6). Reserved keys
   are named here so v1 data stays forward-compatible.
2. **No automatic layout.** Node coordinates are author-provided. Auto-layout is out of scope.
3. **No live data binding.** An atlas is a static snapshot; it does not poll or fetch project state.
4. **No build toolchain dependency.** The Builder is an *optional* single script (Node, zero npm
   deps) and never a hard requirement — the inline/no-runtime path (§8.4) always works; the
   inline-edit path needs no tooling at all.

---

## 2. System overview

An atlas is produced and consumed through four components:

| Component   | Artifact                              | Role                                                                 |
|-------------|---------------------------------------|----------------------------------------------------------------------|
| **Data**    | `*.json` (or embedded `#workflow-data`) | The atlas: meta, viewBox, columns, groups, nodes, flows (§3, §4).  |
| **Engine**  | `assets/template.html`                | Generalized renderer + interaction layer (§6, §7). Theme-agnostic.   |
| **Themes**  | `assets/themes/*.json`                | Named palettes → CSS custom properties + lane colors (§5).           |
| **Builder** | `scripts/atlas` (optional Node CLI)    | Injects Data + Theme into the Engine → self-contained HTML (§8). Optional — see §8.4. |

Two authoring paths are REQUIRED (per product decision):

- **Build path (primary).** `atlas build data.json --theme <name> -o spec.html`. Data lives in its
  own file; the Builder substitutes it and the theme into the template.
- **Inline path (secondary).** Copy `template.html`, edit the embedded `#workflow-data` block and
  the `data-theme` attribute in place. No script required.

Both paths **MUST** produce byte-for-byte equivalent rendering for the same data + theme.

---

## 3. Core domain model

An **Atlas** is the tuple `(meta, viewBox, columns[], groups[], nodes[], flows[])`.

### 3.1 Node — a *state*

A **Node** represents a durable state, component, table, or store — *a thing the system can be in
or hold*, not an action. Each node belongs to exactly one **Group** (its lane) and is positioned
absolutely within the viewBox.

> Examples (from the reference atlas): `documents (status=normalized)`, `document_raw`,
> `entities / document_entities`, `knowledge/`.

### 3.2 Flow — an *operation*

A **Flow** represents an operation that transforms state: a command, job, pipeline, or handoff.
A flow is an **ordered list of Steps**. The order is the reading order — step *i* is read before
step *i+1*. Flows are the primary unit of navigation (§7).

> Examples: `日次 蓄積 (無人)`, `手動 取得+分類`, `蒸留 (正本化)`.

### 3.3 Step — a *directed transition with payload*

A **Step** is `(from, to, passes, note?)`:

- `from`, `to` — node ids. The step asserts "operation moves from state `from` toward state `to`."
- `passes` — REQUIRED short string: *what is carried* across the transition (the payload, the
   command, the contract).
- `note` — OPTIONAL longer annotation (file path, ADR ref, caveat).

Multiple steps **MAY** share the same `(from,to)` pair; the Engine aggregates them onto one
**edge** carrying a multi-step badge (§6.4).

### 3.4 Group — a *lane (category)*

A **Group** is a visual+semantic lane. Every node names a group; the group determines the node's
category color (§5.4). Groups are categories, *not* states (per §1.2.1).

### 3.5 Column — a *layout band*

A **Column** is a labeled vertical band used purely for layout legibility (a header label and an
optional dashed divider x-position). Columns carry no semantics beyond grouping nodes spatially and
**MAY** differ in count/identity from groups.

### 3.6 Invariants

An atlas is **well-formed** only if all hold (the Builder **MUST** validate; the Engine **SHOULD**
degrade gracefully):

1. Every `node.id` is unique.
2. Every `node.group` references an existing `group.id`.
3. Every `step.from` and `step.to` reference an existing `node.id`.
4. Every `flow.id` is unique; every `step.passes` is a non-empty string.
5. `viewBox.w > 0` and `viewBox.h > 0`.
6. A flow has at least one step.
7. `nodes` and `flows` are **non-empty arrays**; `groups` (if present) is an array. (A
   missing/empty `flows` would crash the Engine at `DATA.flows[0]`; the Builder rejects it.)

The Engine **SHOULD** additionally tolerate steps that reference an unknown `node.id` (possible on
the hand-edited inline path) by skipping that edge rather than throwing.

---

## 4. Data format

### 4.1 Container

Data is a single JSON object. In the **build path** it lives in a `.json` file. In the **inline
path** it lives verbatim inside:

```html
<script type="application/json" id="workflow-data"> … </script>
```

The Engine **MUST** load data by: (a) reading `#workflow-data` if present (enables `file://`), else
(b) `fetch("./<name>.json")`. (a) takes precedence so a built file never needs the network.

### 4.2 Schema (informative JSON-Schema-flavored summary)

```jsonc
{
  "meta": {
    "title":  "string",            // REQUIRED — document + <title>
    "repo":   "string?",           // OPTIONAL — project root, shown in header
    "note":   "string?"            // OPTIONAL — legend / ADR pointer
  },
  "viewBox": { "w": "number", "h": "number" },   // REQUIRED — SVG coordinate space

  "columns": [{
    "id":      "string",           // REQUIRED unique
    "label":   "string",           // REQUIRED — header text (letter-spaced)
    "x":       "number",           // REQUIRED — header center x
    "divider": "number | null"     // OPTIONAL — dashed divider x; null = none
  }],

  "groups": [{
    "id":      "string",           // REQUIRED unique — referenced by node.group
    "label":   "string",           // REQUIRED — human name
    // Color is HYBRID (§5.4). All three are OPTIONAL overrides; when omitted the
    // theme assigns this lane a palette color by group order.
    "stroke":  "string?",          // node border / accent
    "fill":    "string?",          // node background
    "sub":     "string?"           // node subtitle text color
  }],

  "nodes": [{
    "id":       "string",          // REQUIRED unique
    "title":    "string",          // REQUIRED — bold line
    "subtitle": "string?",         // OPTIONAL — muted second line
    "group":    "string",          // REQUIRED — group.id
    "x": "number", "y": "number",  // REQUIRED — top-left in viewBox space
    "w": "number", "h": "number"   // REQUIRED — box size
    // RESERVED for v2 (ignored by v1 Engine): "status": "ok|warn|error|running|disabled|skip"
  }],

  "flows": [{
    "id":          "string",       // REQUIRED unique — hash deep-link target
    "icon":        "string?",      // OPTIONAL — emoji/glyph in chip + panel
    "name":        "string",       // REQUIRED — chip + panel title
    "sub":         "string?",      // OPTIONAL — one-line subtitle
    "description": "string?",      // OPTIONAL — panel paragraph
    "steps": [{
      "from":   "string",          // REQUIRED — node.id
      "to":     "string",          // REQUIRED — node.id
      "passes": "string",          // REQUIRED — payload/contract carried
      "note":   "string?"          // OPTIONAL — annotation
    }]
  }]
}
```

### 4.3 Compatibility & ordering

- Flow order is significant: chips and number-key bindings follow array order (flow *n* ⇢ key
  `n`, with the 10th flow bound to `0`; §7.2).
- The Engine **MUST** ignore unknown keys (forward compatibility for v2 fields such as
  `node.status`).
- The Engine **MUST NOT** require any optional field; missing optionals degrade to sensible
  defaults (e.g. missing `icon` → no glyph; missing `subtitle` → single-line node).

---

## 5. Theming specification

### 5.1 Token model

A **theme** is a named JSON object resolving to two things: (a) a set of **chrome tokens** (CSS
custom properties for background, surface, text, borders, the active-edge accent), and (b) an
ordered **lane palette** used to color groups (§5.4). The Engine reads tokens exclusively through
CSS custom properties on `:root`; it **MUST NOT** hardcode any color in script.

```jsonc
{
  "id": "okabe-ito-light",
  "name": "Okabe-Ito (Light)",
  "mode": "light",                 // "light" | "dark" — affects derivations (§5.5)
  "chrome": {
    "bg":            "#F8FAFC",
    "surface":       "#FFFFFF",
    "panel":         "#F1F5F9",
    "line":          "#CBD5E1",
    "grid":          "#E2E8F0",
    "text":          "#111827",
    "muted":         "#64748B",
    "edge":          "#94A3B8",     // faint base edge
    "edge-active":   "#0072B2",     // highlighted flow edge + badge
    "edge-active-glow": "rgba(0,114,178,0.35)"
  },
  "lanes": [                        // ordered palette — assigned to groups by order (§5.4)
    "#0072B2", "#009E73", "#E69F00", "#CC79A7", "#56B4E9", "#D55E00", "#000000"
  ]
}
```

### 5.2 Bundled presets (REQUIRED)

The skill **MUST** ship the following preset themes. Each name is the `--theme` value and the
`data-theme` attribute value:

| `--theme`            | Mode  | Family            | Intended use                                  |
|----------------------|-------|-------------------|-----------------------------------------------|
| `okabe-ito-light`    | light | Okabe-Ito (viz)   | **Default.** Figures, legends, color-vision safe |
| `okabe-ito-dark`     | dark  | Okabe-Ito (viz)   | Same palette on dark chrome                    |
| `catppuccin-latte`   | light | Catppuccin        | UI-cohesive light                             |
| `catppuccin-mocha`   | dark  | Catppuccin        | UI-cohesive dark                              |
| `solarized-light`    | light | Solarized         | Document/JSON/log adjacency                    |
| `solarized-dark`     | dark  | Solarized         | Classic dark                                  |
| `nord`               | dark  | Terminal (Nord)   | Calm blue dark                                |
| `dracula`            | dark  | Terminal (Dracula)| High-vibe dark (matches the original prototype mood) |
| `gruvbox-dark`       | dark  | Terminal (Gruvbox)| Warm retro dark                               |

The **default** theme is `okabe-ito-light` (per §10 the viz palettes win on readability). The
reference prototype's look is preserved as `dracula`/`gruvbox-dark` for users who want it.

> **Lane vs. state separation (§10.3).** Preset lane palettes are *qualitative/categorical* only.
> Sequential/diverging palettes (ColorBrewer, Paul Tol) are reserved for the deferred score/heatmap
> features and **MUST NOT** be used as lane palettes.

### 5.3 Resolution at build / load time

- **Build path:** `atlas build … --theme <name>` inlines the theme's `chrome` as `:root{…}` custom
  properties and stamps `data-theme="<name>"` on `<html>`. No theme lookup happens at runtime.
- **Inline path:** the template embeds all presets as a small `<script id="atlas-themes">` table
  and applies the one named by `data-theme`; switching is changing that attribute.
- The Engine **MUST** apply exactly one theme. If `data-theme` is absent or unknown it **MUST**
  fall back to `okabe-ito-light`.

### 5.4 Hybrid lane coloring (per product decision)

Each group resolves its three node colors `(stroke, fill, sub)` by this precedence:

1. **Explicit override.** If `group.stroke` / `group.fill` / `group.sub` is present in the data, use
   it verbatim. (This was the original prototype behavior and remains fully supported.)
2. **Theme assignment.** Otherwise assign `accent = theme.lanes[i mod lanes.length]`, where `i` is
   the group's index in the `groups[]` array, and **derive** the missing channels (§5.5).

Overrides and assignment compose per-channel: a group MAY set only `stroke` and let `fill`/`sub`
derive. The Engine **MUST** compute this resolution once at load and expose it via the existing
`--gstroke` / `--nsub` custom properties already used by `template.html`.

### 5.5 Channel derivation

When `fill` or `sub` is not explicitly provided, the Engine derives them from `accent`, `surface`,
and `text` so a theme needs only one color per lane:

- `fill  = mix(accent, surface, 0.86)` — i.e. 14% accent over surface (subtle tint).
- `sub   = mix(accent, text, 0.45)`.

`mix(a,b,t)` is linear sRGB interpolation returning `a*(1-t)+b*t`. In `dark` mode `surface` is the
dark panel color, yielding dark tinted fills; in `light` mode it yields pale tinted fills. Themes
**MAY** instead provide explicit per-lane `{stroke,fill,sub}` triples in a `laneColors` array to
override derivation entirely.

### 5.6 Reserved: status colors (v2, non-normative)

v2 will add an optional `node.status` and a theme `status` block mapping
`ok/warn/error/running/disabled/skip` to colors plus a non-color cue (icon, border style, progress
ring) per §10.2. v1 Engines **MUST** ignore `node.status`. The recommended v2 mapping (Okabe-Ito):
`ok=#009E73`, `warn=#E69F00`, `error=#D55E00`, `running=#56B4E9`, `disabled=#94A3B8`. Listed here
only so v1 data authored today stays valid.

---

## 6. Rendering contract

The reference Engine is `assets/template.html`. This section fixes the parts an implementation
**MUST** preserve.

### 6.1 Layer order

The SVG **MUST** contain these `<g>` layers in this z-order (bottom→top): `grid`, `base-edges`,
`flow-edges`, `nodes`, `badges`. Badges are last so step numbers always sit above nodes.

### 6.2 Grid

For each column: a centered header label at the top; if `divider != null`, a dashed vertical line.
Columns are decorative/legibility only and carry no interaction.

### 6.3 Edges

- **Base edges:** the set of all unique directed `(from,to)` pairs across *all* flows, drawn faint.
  Visible only in overview mode (no active flow).
- **Flow edges:** when a flow is active, only that flow's edges are drawn, highlighted, with the
  active-edge accent and arrowheads.
- Edge geometry is a cubic Bézier. Same-column pairs connect top/bottom; cross-column pairs connect
  left/right. A deterministic **lane offset** separates `A→B` from a co-existing `B→A` so reverse
  edges don't overlap. (Reference: `edgeGeom`, `laneFor` in `assets/template.html`.)

### 6.4 Step badges (one badge per edge)

Steps sharing a `(from,to)` edge aggregate to **one badge** labeled with their step numbers
(`"3"`, `"3 · 7"`, or `"3–9 (5)"`). Badge placement **MUST** avoid overlapping nodes and other
badges (collision-avoided offset search along the path; tether drawn to the true midpoint if the
badge wanders). Clicking a multi-step badge cycles through its constituent steps.

### 6.5 Annotation panel

The side panel renders the active flow's `icon`, `name`, `description`, then an ordered list of
steps. Each list item shows `from.title → to.title`, the `passes` payload, and the `note`. Clicking
an item focuses that step (§7.3). The panel is the linear, readable "spec" view; the canvas is the
spatial view. They **MUST** stay in sync (same focused step highlighted in both).

### 6.6 Focus highlight ("PIKAPIKA")

Focusing a step **MUST** simultaneously: pulse its badge, pulse its edge, glow its two endpoint
nodes, and flash its annotation list item, then pan the canvas to the badge. The specific animation
is RECOMMENDED to match the reference engine but MAY be tuned; the four-surface synchronization is required.

---

## 7. Interaction model

### 7.1 Modes

- **Overview** — no active flow: all nodes lit, base edges faint, panel shows a hint.
- **Flow** — one active flow: its nodes lit, others dimmed, its edges + badges drawn, panel lists
  steps.
- **Step focus** — within a flow, one step is the current step (highlighted per §6.6).

### 7.2 Keyboard — flow selection (existing)

- `1`–`9` select flows 1–9; `0` selects flow 10.
- `F` fit/zoom to whole atlas. `+`/`-` zoom. `Esc` clears focus, then (second press) exits the flow
  to overview.

### 7.3 Keyboard — step stepping (NEW, REQUIRED)

This is the primary new capability over the prototype. When a flow is active:

| Key                       | Action                                                            |
|---------------------------|-------------------------------------------------------------------|
| `→` or `↓`                | Focus the **next** step (advance reading order).                  |
| `←` or `↑`                | Focus the **previous** step.                                      |
| `Home`                    | Focus the first step.                                             |
| `End`                     | Focus the last step.                                              |

Behavior requirements:

1. Stepping order is `flow.steps` order (the same numbering shown on badges/panel), **not** edge
   order. This is what a reader expects when "walking the operation."
2. If no step is focused yet, `→`/`↓` focuses step 1 and `←`/`↑` focuses the last step.
3. Stepping **clamps** at the ends by default (does not wrap). A theme/config flag MAY enable
   wrap-around; default is clamp.
4. Each step change runs the full focus highlight (§6.6) including the pan, and scrolls the panel
   item into view.
5. Arrow keys **MUST** `preventDefault` only when a flow is active (so overview-mode page scroll is
   unaffected), and **MUST** be ignored when a modifier (Ctrl/Meta/Alt) is held.
6. When a multi-step edge is the focus target, arrow stepping advances by *step index*, transparently
   moving through the constituent steps of that shared edge in order.

### 7.4 Pointer

Wheel = zoom at cursor; drag = pan; click badge/list-item = focus step; click empty canvas = hide
tooltip. Hovering an edge badge shows a tooltip listing that edge's steps.

### 7.5 Deep linking

`#flow=<id>`, `#flow=<n>` (1-based), or `#<id>` selects a flow on load and on `hashchange`. The
Engine SHOULD additionally accept `#flow=<id>:<step>` to deep-link a specific focused step
(1-based); unknown/out-of-range step indexes are ignored.

---

## 8. Build & distribution

### 8.1 Template substitution markers

`template.html` **MUST** carry two comment-delimited sentinel regions (CSS/JS comments, so the raw
template stays a valid runnable file with placeholder/default content), plus two single-token
targets the Builder rewrites by pattern:

```html
<html lang="ja" data-theme="okabe-ito-light">  <!-- Builder rewrites the data-theme attribute -->

<title>Workflow Atlas</title>                  <!-- Builder rewrites the <title> element -->

<style id="atlas-theme">/*ATLAS:THEME*/ … default :root tokens … /*/ATLAS:THEME*/</style>

<script type="application/json" id="workflow-data">
/*ATLAS:DATA*/ { … reference/placeholder atlas … } /*/ATLAS:DATA*/
</script>
```

The `ATLAS:DATA` markers are stripped by the Engine before `JSON.parse` (JSON forbids comments), so
they persist across builds. The `<title>` and `data-theme` are rewritten by regex, not by sentinel
comments (comments inside `<title>` would render as visible text).

### 8.2 Builder CLI

```
atlas build <data.json> [options]
  --theme <name>     preset theme id (default: okabe-ito-light)
  --out, -o <file>   output path (default: <data basename>.html)
  --title <text>     override meta.title for <title>
  --validate-only    run §3.6 validation and exit (no output)
```

The Builder **MUST**:

1. Parse and **validate** the data against §3.6; on failure exit non-zero with a precise message
   (which invariant, which id/index).
2. Read `template.html`, replace the `ATLAS:DATA` region with the data JSON, the `ATLAS:THEME`
   region with the chosen theme's `:root{…}` tokens, the `data-theme` attribute, and the `<title>`.
3. Emit a single HTML file with **no** external references. The Builder **MUST NOT** require network
   access or package-manager dependencies (Node stdlib only).
4. Be idempotent: re-running on the same inputs yields identical output.
5. **In the embedded data JSON, replace every less-than character with its JSON unicode escape
   (the six characters backslash-u-0-0-3-c)** so values containing `</script>`, `<!--`, or
   `<script` cannot break out of the `<script type="application/json">` block (blank page / XSS).
   `JSON.parse` restores the character on load. Insert the data via a function replacer (not a
   string replacement) so `$` sequences in the data are not interpreted.

The Builder reads the bundled theme table from the template's `#atlas-themes` block (single source),
so themes never drift between Builder and Engine.

### 8.3 Inline path

Copying `template.html` and editing the `#workflow-data` block + `data-theme` attribute **MUST**
yield the same render as the build path for the same data+theme. The default content inside the
sentinels is the reference atlas so a fresh copy is immediately runnable.

### 8.4 Build tiers & the no-runtime fallback (REQUIRED)

The Builder is a **convenience, not a hard dependency**. Conforming distribution provides a tiered
path so an atlas can always be produced regardless of OS or installed runtimes:

1. **Node present (RECOMMENDED).** `node atlas build …` does substitution **and** §3.6 validation in
   one runtime. This is the safest path and is cross-platform (Linux/macOS/**Windows** via
   `node atlas …`). Because the build is real JSON handling (not shell text-munging), it avoids
   quoting/escaping hazards.
2. **No runtime / scripts disallowed / any OS.** The **inline path** (§8.3) is the universal
   fallback: an agent (e.g. Claude) or a human copies `template.html` and edits the embedded
   `#workflow-data` JSON block and the `data-theme` attribute directly. This needs **no** tooling and
   works identically on every platform, including Windows.

A POSIX-shell builder is intentionally **not** provided: robust JSON validation in shell requires a
non-stdlib tool (`jq`), and POSIX shell is not natively available on Windows — both conflict with the
zero-dependency, cross-platform goal. The two tiers above cover every environment.

---

## 9. Skill structure (mirrors `cathrynlavery/diagram-design`)

This repo is a **Claude Code / Codex plugin**. Per the skills spec, a plugin's skills are discovered
under `skills/<skill-name>/SKILL.md`; the plugin root needs **no** top-level `SKILL.md` (that would
register a redundant, name-colliding `workflow-atlas:workflow-atlas` skill). Human landing docs live
in `README.md` / `README.ja.md`.

```
workflow-atlas/
├── SPEC.md                          # this document (normative)
├── README.md / README.ja.md         # human onboarding + install (en + ja)
├── LICENSE                          # MIT
├── .claude-plugin/plugin.json       # Claude Code plugin metadata
├── .codex-plugin/plugin.json        # Codex plugin metadata
├── skills/workflow-atlas/
│   ├── SKILL.md                     # the skill entry point (progressive disclosure)
│   ├── references/                  # load-on-demand docs
│   │   ├── schema.md                # §4 expanded with worked field examples
│   │   ├── themes.md                # §5 + full preset palette tables
│   │   ├── authoring.md            # how to model state×operation; node placement tips
│   │   ├── keybindings.md           # §7 reader reference
│   │   └── build.md                 # §8 Builder usage + inline/no-runtime path
│   ├── scripts/
│   │   └── atlas                     # optional Builder (Node, executable, zero deps)
│   └── assets/
│       ├── template.html            # the Engine + sentinels (generic; edit data only)
│       ├── themes/*.json             # one file per preset (§5.2)
│       └── examples/                # one data file per language (en + ja)
│           ├── rag-agent.{en,ja}.json     # generic reference atlas
│           └── hermes-agent.{en,ja}.json  # real repo: NousResearch/hermes-agent
└── docs/                            # GitHub Pages gallery (bilingual)
    ├── index.html                   # gallery landing (EN/JA toggle; links every build)
    ├── rag-agent.{en,ja}.html       # built atlases
    └── hermes-agent.{en,ja}.html
```

**Progressive disclosure (per diagram-design):** `SKILL.md` always loads; each `references/*.md`
loads only when its topic is invoked, keeping context tight. `SPEC.md` is the single source of
truth; reference docs MUST NOT contradict it.

---

## 10. Accessibility & color guidance (normative where marked)

Derived from the color research (Okabe-Ito / CUD / WCAG / ColorBrewer / Paul Tol).

1. **Contrast.** Node title text on its fill **MUST** meet WCAG AA (≥ 4.5:1 normal, ≥ 3:1 large).
   Theme authors **MUST** verify this; the Builder **SHOULD** warn on violations.
2. **No meaning by color alone.** Category is conveyed by lane color **and** column position **and**
   label. Step identity is conveyed by the number badge, not color. (v2 status will add icon/border
   cues, not color alone.)
3. **Category ≠ state.** Lane (category) palettes are qualitative; score/heat (v2) uses sequential/
   diverging palettes. The two **MUST NOT** share a color system.
4. **Thin/small elements.** Pale palette colors (e.g. Okabe-Ito yellow `#F0E442`) **MUST NOT** be
   used for thin lines or small text; restrict them to filled highlight surfaces.
5. **Default to light + viz palette.** The default theme is `okabe-ito-light` because document/JSON/
   log adjacency and figure legibility outrank terminal aesthetics for this artifact's purpose.

---

## 11. Conformance — Definition of Done

An implementation conforms to Draft v1 when:

- [ ] **Engine** renders the reference atlas from §Appendix A (layers, edges, badges, panel) under
      the default theme.
- [ ] **Engine** supports arrow/Home/End step navigation per §7.3, synced across canvas + panel.
- [ ] **Engine** resolves lane colors via the §5.4 hybrid precedence and reads all color from CSS
      custom properties only.
- [ ] **Engine** falls back to `okabe-ito-light` on missing/unknown `data-theme`.
- [ ] **Builder** validates §3.6, substitutes the `ATLAS:DATA`/`ATLAS:THEME` regions + `data-theme`
      + `<title>`, emits a dependency-free single file, and is idempotent.
- [ ] **No-runtime path** (§8.4) is documented: the inline edit of `template.html` works on any OS
      with no tooling and renders identically to the built file for the same data + theme.
- [ ] All nine preset themes (§5.2) exist, each passing the §10.1 contrast check on node titles.
- [ ] A fresh `template.html` (unbuilt) is itself a runnable atlas showing the reference data.
- [ ] `SKILL.md` + `references/*.md` exist and do not contradict this SPEC.

---

## Appendix A — Reference atlas

The canonical example is the generic `AI Agent — RAG pipeline (state × operation)` atlas:
6 columns, 6 groups, 14 nodes, 6 flows (index → retrieve → generate → tool-loop → memory →
deliver). It is repository-agnostic on purpose (no personal/private project data) and ships in two
languages: `assets/examples/rag-agent.en.json` and `rag-agent.ja.json`. The English one is also the
default embedded in `template.html`, so a fresh unbuilt template renders it immediately.

A second example models a real OSS codebase — `NousResearch/hermes-agent` — traced from source with
`file:line` citations: 5 columns, 5 groups, 23 nodes, 19 flows
(`assets/examples/hermes-agent.{en,ja}.json`). Both examples are published under the default
`okabe-ito-light` theme as `docs/<name>.<lang>.html`, indexed by the bilingual gallery
`docs/index.html`. All examples are color-free at the group level so every preset theme recolors
them (§5.4).

## Appendix B — Preset lane palettes (informative)

| Theme              | Lane palette (ordered)                                                            |
|--------------------|----------------------------------------------------------------------------------|
| okabe-ito-*        | `#0072B2 #009E73 #E69F00 #CC79A7 #56B4E9 #D55E00 #000000`                          |
| catppuccin-latte   | `#1e66f5 #40a02b #df8e1d #ea76cb #04a5e5 #d20f39 #8839ef` (Latte blue/green/yellow/pink/sky/red/mauve) |
| catppuccin-mocha   | `#89b4fa #a6e3a1 #f9e2af #f5c2e7 #89dceb #f38ba8 #cba6f7`                          |
| solarized-*        | `#268bd2 #859900 #b58900 #d33682 #2aa198 #dc322f #6c71c4` (blue/green/yellow/magenta/cyan/red/violet) |
| nord               | `#81a1c1 #a3be8c #ebcb8b #b48ead #88c0d0 #bf616a #5e81ac`                          |
| dracula            | `#bd93f9 #50fa7b #f1fa8c #ff79c6 #8be9fd #ff5555 #ffb86c`                          |
| gruvbox-dark       | `#83a598 #b8bb26 #fabd2f #d3869b #8ec07c #fb4934 #fe8019`                          |

Chrome tokens for each preset are defined in `assets/themes/<id>.json` (§5.1).

---

*End of Draft v1.*
