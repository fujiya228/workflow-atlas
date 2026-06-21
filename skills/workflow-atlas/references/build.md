# Build reference

Producing a built atlas has **two tiers**, so it works on any OS regardless of installed runtimes.
Both render identically. Authoritative: `SPEC.md` §8.

| Tier | When | How | Validation |
|------|------|-----|------------|
| **A — Node CLI** (recommended) | Node is available | `node scripts/atlas build …` | full §3.6 check |
| **B — inline / no-runtime** (universal) | no Node, scripts disallowed, **Windows**, or an agent is doing it | edit a copy of `template.html` directly | by reading the schema |

The Node CLI is a **convenience, not a hard dependency**. If you can't (or don't want to) run it,
Tier B always works — including Windows, where you simply edit the file.

## Tier A — Node CLI

```bash
node scripts/atlas build <data.json> [options]
```

| Option              | Default                | Meaning                                  |
|---------------------|------------------------|------------------------------------------|
| `--theme <name>`    | `okabe-ito-light`      | preset id (see `references/themes.md`)    |
| `-o, --out <file>`  | `<data basename>.html` | output path                              |
| `--title <text>`    | `meta.title`           | override the `<title>`                    |
| `--validate-only`   | —                      | validate and exit, write nothing          |
| `-h, --help`        | —                      | usage                                    |

```bash
# default theme, output next to the data file
node scripts/atlas build assets/examples/rag-agent.en.json

# pick a theme + explicit output
node scripts/atlas build assets/examples/rag-agent.en.json --theme dracula -o /tmp/atlas.html

# just check the data is well-formed (no output written)
node scripts/atlas build assets/examples/rag-agent.en.json --validate-only
```

What it does: validates the data (SPEC §3.6), then substitutes the template's `ATLAS:DATA` and
`ATLAS:THEME` regions, the `<html data-theme="…">` attribute, and `<title>`. Output is one
self-contained HTML file — no network, no external assets, byte-identical on re-run (idempotent).
Node stdlib only; no `npm install`.

**Cross-platform:** invoke as `node scripts/atlas …` on Linux/macOS/**Windows** (identical). Why
Node and not a shell script: the build is real JSON handling, and doing it in one runtime avoids
shell quoting/escaping hazards; POSIX shell also isn't native on Windows and robust JSON validation
in shell needs `jq`. So: Node when present, else Tier B.

## Tier B — inline edit (no tooling, any OS, agent-friendly)

This is the universal fallback and what an agent (e.g. Claude) does when no runtime is available.

```bash
cp assets/template.html spec.html      # or copy in any file manager (Windows ok)
```

Then edit `spec.html`:

1. Replace the JSON between the `/*ATLAS:DATA*/ … /*/ATLAS:DATA*/` markers inside
   `<script type="application/json" id="workflow-data"> … </script>` with your atlas. Leave the
   markers in place (they're stripped at load; JSON itself can't contain comments).
2. Set the theme on the root element: `<html lang="ja" data-theme="catppuccin-mocha">`.

A fresh copy of `template.html` is already a runnable atlas showing the sample data — open it first,
then edit. The reader can still switch themes via the header dropdown regardless of `data-theme`.

> **Doing it as an agent:** copy the template, then make exactly the two edits above. Validate the
> data against `references/schema.md` / `SPEC.md` §3.6 before saving (unique ids; every
> `step.from/to` is a real node; non-empty `passes`; `viewBox.w/h > 0`).

## Validation errors (Tier A)

`--validate-only` (and every build) reports the exact offender for:

- missing `meta.title`; non-positive `viewBox.w/h`
- duplicate `node.id` / `flow.id`
- `node.group` referencing an unknown group
- `step.from` / `step.to` not matching any node id
- empty `step.passes`
- a flow with no steps

Exit codes: `0` ok · `1` usage/IO error · `2` data validation failed.

## Publishing (GitHub Pages)

Build each atlas into `docs/` and list it in `docs/index.html` (the gallery). With Pages set to
*Source: `main` / `docs`*, every built atlas is served as a static page. Example:

```bash
node scripts/atlas build assets/examples/rag-agent.en.json -o ../../docs/rag-agent.en.html
```
