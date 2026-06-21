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

## 2. Group nodes into lanes (categories)

A **group** is a category that shares a color: `External`, `Ingest`, `Enrich`, `LLM`, `Derive`,
`Knowledge`. Pick 4–8 groups. Groups should be *kinds of state*, not *statuses*. Order them
left→right in the order data tends to flow; the theme assigns lane colors by this order, so the
left-most group gets palette color 0.

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

- [ ] Every node is a noun/state; every flow is a verb/operation.
- [ ] 4–8 groups, ordered to match data flow.
- [ ] Columns labeled, dividers placed, nodes on a tidy grid with top headroom.
- [ ] Flow names numbered; steps in reading order; `passes` short and concrete.
- [ ] `node scripts/atlas build data.json --validate-only` passes.
