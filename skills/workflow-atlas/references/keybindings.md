# Reader controls

How to read a built atlas in the browser. Authoritative: `SPEC.md` §7.

## Keyboard

| Key                | Action                                                              |
|--------------------|--------------------------------------------------------------------|
| `1`–`9`            | Select flow 1–9                                                     |
| `0`                | Select flow 10                                                      |
| `→` or `↓`         | **Next step** of the active flow (advance reading order)            |
| `←` or `↑`         | **Previous step**                                                   |
| `Home`             | First step                                                          |
| `End`              | Last step                                                           |
| `F`                | Fit / zoom to the whole atlas                                       |
| `+` / `-`          | Zoom in / out (centered)                                            |
| `Esc`              | 1st press: clear step focus · 2nd press: exit flow → overview       |

**Stepping notes**

- Stepping walks the flow's **step order** (the numbers on badges and in the panel), not edge order.
- With no step focused yet, `→`/`↓` jumps to step 1 and `←`/`↑` to the last step.
- Stepping **clamps** at the ends (does not wrap).
- Each step change runs the full highlight (badge + edge + endpoint nodes + panel item) and pans
  the canvas to that step; the panel item scrolls into view.
- Arrow keys only act (and only block page scroll) while a flow is active; in overview they do
  nothing. Keys are ignored while a modifier is held or focus is in the theme dropdown.

## Mouse / pointer

| Action                     | Result                                          |
|----------------------------|-------------------------------------------------|
| Click a flow chip          | Select that flow                                |
| Click a step badge         | Focus that step (multi-step badge cycles)       |
| Click a panel step item    | Focus that step                                 |
| Hover a badge              | Tooltip listing that edge's step(s)             |
| Wheel                      | Zoom at cursor                                  |
| Drag canvas                | Pan                                             |
| Theme dropdown (header)    | Switch among the 9 color schemes live           |
| Zoom buttons (bottom-right)| `＋` in · `－` out · `⤢` fit                     |

## Deep links (URL hash)

| Hash                    | Opens                                  |
|-------------------------|----------------------------------------|
| `#flow=<id>`            | that flow                              |
| `#flow=<n>`             | the n-th flow (1-based)                |
| `#<id>`                 | that flow                              |
| `#flow=<id>:<step>`     | that flow with step `<step>` focused (1-based) |

Hash changes are live — linking to `#flow=distill:3` jumps straight to step 3 of the distill flow.
