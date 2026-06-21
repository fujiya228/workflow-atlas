# Workflow Atlas

*日本語: [README.ja.md](README.ja.md)*

**Specify a project's state, operations, and flows as JSON — and read them as a single
interactive HTML specification.** Not a diagram you draw; a spec you walk.

A *node* is a **state** (a table, store, component, actor — something the system is in or holds).
A *flow* is an **operation** (an ordered list of steps that transforms state). You write only JSON;
a generic engine renders one self-contained, offline `.html` with keyboard-walkable steps and nine
swappable color schemes.

[![Workflow Atlas — pick an operation, step through it with the arrow keys (canvas + panel stay in sync), then switch color scheme](docs/media/demo.gif)](https://fujiya228.github.io/workflow-atlas/)

**▶ Live gallery:** https://fujiya228.github.io/workflow-atlas/

## Why

Hand-drawn / mermaid / graphviz diagrams answer *"what connects to what"* and rot when the system
changes. A Workflow Atlas answers *"which operation moves the system from state A to state B, and in
what order do the steps run"* — and stays a single editable JSON document.

## Install

**Clone + symlink** (use as a Claude Code skill):

```bash
git clone https://github.com/fujiya228/workflow-atlas ~/code/workflow-atlas
ln -s ~/code/workflow-atlas/skills/workflow-atlas ~/.claude/skills/workflow-atlas
```

**Plugin marketplace** (Claude Code):

```
/plugin marketplace add fujiya228/workflow-atlas
/plugin install workflow-atlas@workflow-atlas
```

No runtime dependencies. The build script needs only Node (stdlib); the inline path needs nothing.

## Use

```bash
cd skills/workflow-atlas

# build the bundled example (Tier A — needs Node; cross-platform incl. Windows)
node scripts/atlas build assets/examples/rag-agent.en.json --theme okabe-ito-light -o /tmp/atlas.html

# author your own
cp assets/examples/rag-agent.en.json my.json   # edit my.json to your system
node scripts/atlas build my.json --theme dracula -o my.html
```

No Node / on Windows / building as an agent? Copy `assets/template.html` and edit its embedded
`#workflow-data` JSON block + `data-theme` directly — no tooling, any OS (see
[`references/build.md`](skills/workflow-atlas/references/build.md)).

Open the HTML and:

- **number keys `1`–`0`** pick an operation (flow),
- **arrow keys `← →` / `↑ ↓`** step through it (`Home`/`End` jump to ends),
- **theme dropdown** flips among all nine color schemes,
- **wheel/drag** zoom and pan, **`F`** fits, **`Esc`** backs out.

Prefer no tooling? Copy `assets/template.html` and edit the embedded `#workflow-data` JSON block
directly — it's a runnable atlas out of the box.

## Color schemes

Nine presets, default `okabe-ito-light` (a color-vision-safe visualization palette — chosen for
readability over terminal aesthetics): Okabe-Ito (light/dark), Catppuccin (Latte/Mocha), Solarized
(light/dark), Nord, Dracula, Gruvbox Dark. See
[`references/themes.md`](skills/workflow-atlas/references/themes.md).

## Documentation

- [`SPEC.md`](SPEC.md) — normative specification (the JSON schema, theming, rendering, keybindings,
  build, conformance).
- [`skills/workflow-atlas/SKILL.md`](skills/workflow-atlas/SKILL.md) — the skill entry point.
- `skills/workflow-atlas/references/` — schema, authoring, themes, keybindings, build.

## License

MIT — see [LICENSE](LICENSE).
