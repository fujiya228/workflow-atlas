# Data schema reference

The atlas data is a single JSON object. Authoritative definition: `SPEC.md` §3–§4. This is the
working reference with examples. Start by copying `assets/examples/rag-agent.en.json`.

## Top-level shape

```jsonc
{
  "meta":    { … },   // document metadata
  "viewBox": { … },   // SVG coordinate space
  "columns": [ … ],   // layout bands (decorative)
  "groups":  [ … ],   // lanes / categories (color)
  "nodes":   [ … ],   // states
  "flows":   [ … ]    // operations (ordered steps)
}
```

## `meta`

| Field   | Req | Notes                                              |
|---------|-----|----------------------------------------------------|
| `title` | ✔   | Document title + `<title>` + header brand          |
| `repo`  |     | Project root path, shown muted in the header       |
| `note`  |     | Legend / ADR pointer (free text)                   |
| `lang`  |     | UI-chrome language: `"en"` (default) or `"ja"`. Localizes the hint, zoom tooltips, overview text, etc., and sets `<html lang>`. Node/flow content language is whatever you write. |

## `viewBox`

```json
{ "w": 1600, "h": 900 }
```

The SVG coordinate space all node `x/y/w/h` and column `x` live in. Both **MUST** be > 0. Node
coordinates are absolute within this box. The canvas auto-fits this box on load and on `F`.

## `columns[]` — layout bands (no semantics)

```json
{ "id": "ingest", "label": "INGEST", "x": 400, "divider": 530 }
```

| Field     | Req | Notes                                                        |
|-----------|-----|--------------------------------------------------------------|
| `id`      | ✔   | unique                                                       |
| `label`   | ✔   | header text (rendered letter-spaced, centered at `x`)        |
| `x`       | ✔   | header center x                                             |
| `divider` |     | x of a dashed vertical divider; `null` = none                |

Columns are purely for legibility. They may differ in count/identity from `groups`.

## `groups[]` — lanes / categories (color)

```jsonc
{ "id": "enrich", "label": "Enrich" }                 // theme assigns the color
{ "id": "enrich", "label": "Enrich", "stroke": "#5bc8a0" }  // explicit override
```

| Field    | Req | Notes                                                            |
|----------|-----|------------------------------------------------------------------|
| `id`     | ✔   | unique; referenced by `node.group`                               |
| `label`  | ✔   | human name                                                       |
| `stroke` |     | node border/accent — **override**; omit to let the theme assign  |
| `fill`   |     | node background — override; omit to derive from accent            |
| `sub`    |     | subtitle text color — override; omit to derive                    |

**Hybrid coloring (per channel):** if you omit `stroke/fill/sub`, the selected theme assigns this
lane a palette color by the group's **order** in `groups[]` and derives the tint/subtitle. Provide
any channel to override just that channel. Keep most groups color-free so theme switching works;
override only when a specific lane must be a fixed brand color. Details: `references/themes.md`.

## `nodes[]` — states

```json
{ "id": "documents", "title": "documents", "subtitle": "status=normalized",
  "kind": "store", "group": "ingest", "x": 300, "y": 350, "w": 200, "h": 56 }
```

| Field      | Req | Notes                                            |
|------------|-----|--------------------------------------------------|
| `id`       | ✔   | unique; referenced by steps                      |
| `title`    | ✔   | bold first line                                  |
| `subtitle` |     | muted second line (omit → single-line node)      |
| `kind`     |     | subject type → left-shoulder glyph. One of `trigger` `actor` `process` `decision` `store` (see below). Omit → no glyph. |
| `group`    | ✔   | a `group.id`                                     |
| `x`,`y`    | ✔   | top-left in viewBox space                        |
| `w`,`h`    | ✔   | box size (reference atlas uses `200 × 56`)       |
| `status`   |     | **RESERVED for v2, ignored by v1.** Don't rely on it. |

**`kind` — the subject type (orthogonal to `group`).** `group`/color answers *which lane
(responsibility)*; `kind`/glyph answers *what kind of subject*. Closed set:

| `kind`     | glyph      | meaning                                                       |
|------------|------------|---------------------------------------------------------------|
| `trigger`  | ⚡ bolt     | the start — human instruction or scheduler/event (one kind)   |
| `actor`    | 👤 person  | external party / service                                      |
| `process`  | ⚙ gear     | deterministic, fixed program step                             |
| `decision` | ◇ diamond  | LLM / non-deterministic judgment                              |
| `store`    | ⛁ cylinder | persisted data — DB / file / index / buffer                   |

The glyph inherits the lane's accent color and shifts the title text right to make room. An invalid
`kind` is a build error; an omitted one simply draws no glyph. Most nodes are `store` — reserve the
other four for the boxes whose subject would otherwise be ambiguous.

Layout tip: keep `x/y/w/h` on a 4px (ideally 20px) grid; place each group's nodes in a vertical
stack within its column band. See `references/authoring.md`.

## `flows[]` — operations

```jsonc
{
  "id": "daily-accumulate", "icon": "🌅", "name": "1. 日次 蓄積 (無人)",
  "sub": "timer → 収集 → rule分類", "description": "…一段落の説明…",
  "steps": [
    { "from": "timer", "to": "ingest", "passes": "scripts/daily.sh 起動", "note": "systemd 06:00" },
    { "from": "ingest", "to": "documents", "passes": "本文抽出", "note": "status=normalized" }
  ]
}
```

| Field         | Req | Notes                                                      |
|---------------|-----|------------------------------------------------------------|
| `id`          | ✔   | unique; deep-link target (`#flow=<id>`)                    |
| `name`        | ✔   | chip + panel title. Prefix with `1.` `2.` … to match keys   |
| `icon`        |     | emoji/glyph in chip + panel                                |
| `sub`         |     | one-line subtitle                                          |
| `description` |     | one paragraph shown atop the step list                     |
| `steps[]`     | ✔   | ordered; ≥1                                                |

### `steps[]`

| Field    | Req | Notes                                                          |
|----------|-----|----------------------------------------------------------------|
| `from`   | ✔   | a `node.id` — the state the operation moves *from*             |
| `to`     | ✔   | a `node.id` — the state it moves *toward*                      |
| `passes` | ✔   | **non-empty** — what is carried (payload/command/contract)     |
| `note`   |     | longer annotation (file path, ADR, caveat)                    |

**Order is meaning.** Step *i* is read before *i+1*; arrow keys walk this order. Steps that share a
`(from,to)` pair collapse onto one edge with a combined badge (`"3 · 7"`), and clicking cycles them.

## Ordering & forward-compat

- Flow array order drives chip order and number keys: flow *n* → key `n`; the 10th → `0`. Keys
  beyond 10 flows are unbound (chips/clicks still work).
- Unknown keys are ignored by the engine (forward-compatible with v2 fields like `node.status`).
- Every optional field degrades gracefully when omitted.

## Validate before shipping

```bash
node scripts/atlas build data.json --validate-only
```

Prints `… is valid (N nodes, M flows)` or a precise list of which invariant/id/index failed.
