---
name: workflow-atlas
description: >-
  Turn a project's state nodes and operation flows into a single self-contained
  interactive HTML specification you read (not draw). Use when the user wants to
  document/visualize a system as "data states × operations", walk an operation's
  steps, or produce a portable interactive spec from JSON. Trigger words: workflow
  atlas, state×operation, interactive spec, ワークフロー図, データ状態 × 操作.
---

# Workflow Atlas

A **Workflow Atlas** is an interactive specification you *read*, not a diagram you *draw*.
You declare a project's **state nodes** and the **operations (flows)** that transform between
them as JSON; a generic HTML engine renders it as one self-contained, offline file with
keyboard-walkable steps and swappable color schemes.

**The author edits only JSON.** The engine, themes, and interactions are generic and never need
editing. `SPEC.md` (repo root) is the normative source of truth; this file is the working guide.

## Core model (memorize this)

- **node = a subject** (a thing, not an event). A durable state the system *is in* or *holds*, or
  the entity that owns one — a table, store, component, file, queue, actor, processing step.
  Belongs to one **group** (lane / category) and optionally declares a **kind** (its subject type).
- **flow = an operation.** A command/job/pipeline/handoff that transforms state. An ordered list
  of **steps** — the order is the reading order.
- **step = a directed transition `(from → to)` carrying `passes`** (the payload/contract) and an
  optional `note`. Steps sharing the same `(from,to)` collapse to one numbered badge.
- **group = a category lane** (color). **column = a layout band** (header + optional divider).

If you catch yourself making a node a verb ("send email"), it's probably a *flow step*, not a node.
Nodes are subjects; flows are verbs.

### node.kind — make the subject legible (optional but recommended)

A box alone can't tell the reader *what kind of thing it is* — an actor? a program step? an LLM
judgment? stored data? Set `kind` to render a small glyph at the node's left shoulder. The five
values (a closed set) and their glyphs:

| `kind`     | glyph | use for                                              |
|------------|-------|------------------------------------------------------|
| `trigger`  | ⚡ bolt    | the start: a human instruction **or** a scheduler/event (one kind — don't split them) |
| `actor`    | 👤 person  | an external party or service the system talks to     |
| `process`  | ⚙ gear     | a deterministic, fixed program step                  |
| `decision` | ◇ diamond  | an LLM / non-deterministic judgment                  |
| `store`    | ⛁ cylinder | persisted data — DB, file, index, buffer             |

`kind` is **orthogonal to `group`**: color says *which lane* (responsibility), the glyph says *what
subject type*. Most nodes in a system are `store`; the few `decision`/`process`/`actor`/`trigger`
glyphs are what carry the reading. Omit `kind` if a node doesn't cleanly fit — it just draws no glyph.

### Lanes (groups): divide by responsibility, never by time

The single most common modeling mistake is splitting lanes **per processing step / per output**, so
the lanes become a left-to-right timeline. Don't. The arrows (flows) already carry time and order —
if the lanes also encode time, you've drawn the same axis twice, every lane looks structurally
identical, and the subject of each box becomes unreadable.

Pick a lane axis that is **orthogonal to the arrows** — a *static* partition that doesn't move as
the run progresses:

- **by responsibility / architectural layer** — `External / Entry / Core / Tools / State` (best default; see `hermes-agent` example)
- **by subsystem or team ownership**
- **by subject kind** — if you'd rather make the `kind` axis the lanes themselves

> **Self-check — is my lane axis right?**
> - 🔴 Arrows flow almost entirely left→right in one direction → lanes have become *time*. Re-cut them.
> - 🟢 Arrows criss-cross between lanes in both directions (External→Core→State→Core…) → lanes are *responsibility*, orthogonal to flow. Good.

## Workflow (what to do when invoked)

1. **Model the system** as nodes (states) and flows (operations). See `references/authoring.md`.
2. **Write the data JSON** to the schema in `references/schema.md`. Start from
   `assets/examples/rag-agent.en.json`. Lay nodes out on a grid (multiples of 4; columns left→right).
3. **Build the HTML** (two tiers, pick by environment — see `references/build.md`):
   - **Tier A — Node CLI (if Node is available):** `node scripts/atlas build data.json --theme okabe-ito-light -o spec.html` (validates + builds; works on Linux/macOS/Windows via `node …`).
   - **Tier B — inline edit (no runtime / Windows / when *you* the agent build it):** copy
     `assets/template.html`, then edit the `#workflow-data` JSON block (between the `/*ATLAS:DATA*/`
     markers) and the `<html data-theme="…">` attribute in place. No tooling, any OS. This is the
     universal fallback — when no Node is present, **do this edit yourself** rather than requiring a
     script.
4. **Pick a theme** (`references/themes.md`). Default `okabe-ito-light` (readability-first). The
   built file also has a live theme dropdown, so the reader can switch among all 9 presets.
5. **Open `spec.html`** in a browser. Walk flows with number keys; step with arrow keys.

> **Standing rule — no runtime, no problem.** The Node CLI is optional. If Node isn't available
> (or you're on Windows, or scripts are disallowed), **you build the atlas by editing files
> directly**: copy `assets/template.html` and replace the `#workflow-data` JSON block + the
> `data-theme` attribute yourself (Tier B). Never tell the user a build is impossible for lack of
> Node — fall back to the inline edit.

## Reference files (load on demand)

| File                        | When to read it                                            |
|-----------------------------|-----------------------------------------------------------|
| `references/schema.md`      | Writing/validating the data JSON — every field, with examples |
| `references/authoring.md`   | Deciding what is a node vs a flow; node layout/placement tips |
| `references/themes.md`      | Choosing/overriding colors; the 9 presets; hybrid coloring |
| `references/keybindings.md` | The reader's keyboard/mouse controls                       |
| `references/build.md`       | `atlas build` CLI, the inline path, validation errors      |

## Invariants the data must satisfy

Unique `node.id` and `flow.id`; every `node.group` is a real group; every `node.kind` (if set) is
one of the five values; every `step.from`/`step.to` is a real node id; every `step.passes` is
non-empty; `viewBox.w/h > 0`; each flow has ≥1 step.
`atlas build --validate-only data.json` checks all of these and prints precise errors.

## Constraints (style)

- Self-contained always: no external assets, no network. Renders from `file://`.
- Color carries category, **never** the only signal — column position + label also encode it.
- Lane palettes are categorical (qualitative). Do not repurpose them for scores/heat (that's a
  deferred v2 sequential-palette feature).
- v1 has **no per-node status colors** (success/error/running). `node.status` is reserved and
  ignored; don't rely on it yet.
