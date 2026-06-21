# Authoring guide — modeling state × operation

The hard part of an atlas is not the JSON; it's the **model**. This guide is how to decide what is
a node, what is a flow, and where things go.

## 1. Nodes are nouns, flows are verbs

| Ask                                              | If yes →            |
|--------------------------------------------------|---------------------|
| Is it a thing the system *holds* or *is in*?     | **node** (a state)  |
| Is it an *action* that changes one state to another? | **flow step**   |

Concretely:

- **Nodes:** `documents (normalized)`, `document_raw`, `entities`, `knowledge/`, an external actor
  (`Claude`, `人 / 目的`, `systemd timer`), a queue, a cache, a table, a file tree.
- **Flows:** `日次 蓄積`, `手動 取得+分類`, `蒸留`, `調査 (読み取り)`. Each is one operation a user
  or scheduler triggers, decomposed into ordered steps.

A read-only operation is still a flow — its last step often returns to the human/actor node with
"no writeback" in the `note` (see the reference atlas flow #8).

## 2. Group nodes into lanes — by responsibility, never by time

A **group** is a category that shares a color. Pick 4–8. The axis you divide on decides whether the
atlas reads clearly or turns to mush.

**The one rule: the lane axis must be orthogonal to the arrows.** Flows (arrows) already encode
time and order. If your lanes *also* encode time — i.e. you make one lane per processing step or per
output produced — you've drawn the time axis twice. The symptoms: arrows all march left→right, every
lane looks structurally identical (a process + its data), and you can no longer tell what *kind* of
subject each box is. This is the most common authoring mistake.

Instead pick a **static** partition — one that doesn't move as a run progresses:

| Lane axis                              | Example                                  | Verdict |
|----------------------------------------|------------------------------------------|---------|
| responsibility / architectural layer   | `External / Entry / Core / Tools / State` (hermes) | ◎ best default |
| subsystem / team ownership             | `service-a / service-b / shared`         | ◎ |
| subject kind                           | `Triggers / Actors / Processes / Stores` | ○ (folds the `kind` axis into lanes) |
| data-flow phase                        | `Input / Index / Retrieve / Output` (rag) | △ drifts toward time at the ends |
| **time / processing step**             | `Step 1 / Step 2 / Step 3`               | ✗ collides with the arrows |

> **Self-check.** 🔴 Arrows flow almost entirely left→right → lanes have become *time*; re-cut them.
> 🟢 Arrows criss-cross between lanes in both directions → lanes are orthogonal to flow. Good.

Order groups left→right in the order data tends to flow; the theme assigns lane colors by this
order, so the left-most group gets palette color 0.

## 2b. Tag each node's subject with `kind`

Lanes (color) tell the reader *which responsibility* a box belongs to. They do **not** tell the
reader *what kind of subject* it is. Add `kind` for that — it draws a glyph at the left shoulder and
is fully orthogonal to `group`:

| `kind`     | for                                                  |
|------------|------------------------------------------------------|
| `trigger`  | the start — a human instruction **or** a scheduler/event (don't split these into two) |
| `actor`    | an external party or service                          |
| `process`  | a deterministic, fixed program step                  |
| `decision` | an LLM / non-deterministic judgment                  |
| `store`    | persisted data — DB, file, index, buffer             |

Most nodes are `store`; the handful of non-store glyphs are what make the diagram scannable. If a
node doesn't cleanly fit one value, omit `kind` rather than forcing it.

## 3. Lay out the grid

- One **column** per stage, left→right. Give each a short uppercase `label` and an `x`. Add a
  `divider` between columns for legibility (omit on the last).
- Stack each group's nodes vertically inside its column band. Reference atlas: nodes are `200×56`,
  columns ~260px apart, rows ~190–230px apart, in a `1600×900` viewBox.
- Keep every `x/y/w/h` on a tidy grid (multiples of 4, ideally 20). Even spacing reads as
  intentional; jitter reads as noise.
- Leave headroom at top (`y ≥ 110`) so column headers (drawn at `y≈42`) don't collide.

## 4. Write flows as ordered steps

- Number flow names (`"1. …"`, `"2. …"`) so they line up with number-key bindings.
- Each step's `passes` is the **payload or contract** crossing the edge — keep it short and
  concrete (`"RSS 差分 (max-items 50)"`, `"document_entities 洗い替え"`). Put the command, file,
  or ADR in `note`.
- Order steps in reading order. The reader walks them with arrow keys, so step 1 should be the
  natural entry point (often a trigger: timer/human/source).
- If two steps share the same `from→to`, that's fine — they share one badge and cycle on click.

## 5. Keep it readable

- Color encodes category, but **never alone**: column position and the group label also encode it.
  This matters for color-vision accessibility (see `references/themes.md`).
- Aim for ≤ ~30 nodes and ≤ ~12 flows per atlas. Beyond that, split into multiple atlases (e.g.
  one per subsystem) rather than one dense canvas.
- Prefer the default `okabe-ito-light` theme when the atlas sits next to docs/JSON/logs; it is the
  most legible. Switch to a dark terminal theme only for vibe/presentation.

## 6. Checklist before building

- [ ] Every node is a subject/state; every flow is a verb/operation.
- [ ] 4–8 groups divided by **responsibility, not time** — arrows criss-cross lanes (not all left→right).
- [ ] `kind` set on nodes whose subject would otherwise be ambiguous (trigger/actor/process/decision/store).
- [ ] Columns labeled, dividers placed, nodes on a tidy grid with top headroom.
- [ ] Flow names numbered; steps in reading order; `passes` short and concrete.
- [ ] `node scripts/atlas build data.json --validate-only` passes.
