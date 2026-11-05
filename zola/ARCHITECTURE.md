# AGENTS.md — how an AI should read and build on this repo

This file is for any coding agent (Claude Code, Codex, Cursor, Copilot, a local model)
that clones this repository. Read it first; it is shorter than guessing.

## What this is

ZOLA is a personal operating system: one local web app where every domain of a life
(finance, career, creative practice, intel, the engine itself) is a node orbiting a
central core, each node a small working application, with a deterministic decision
engine underneath. This repository is the **public showcase cut** of that system.

- The live showcase is `world.html`, deployed at
  https://11vinaypathak-sys.github.io/zola-showoff/world.html
- The working shell (`zola.html`) and server (`zola-server.py`) are included as the
  reference implementation. They expect a local server and local data, so on GitHub
  Pages they render as empty shells. That is expected.
- No personal data is in this repo. Real data lives in `*.local.json` files that exist
  only on the owner's machine. The `.gitignore` is an allowlist: everything is ignored
  unless whitelisted.

## The governing idea

**Deterministic core, agentic shell.** Every rule, threshold, scheduler tick and verdict
is plain code you can read and audit. The AI layer reads everything, proposes anything,
and decides nothing. When you extend ZOLA, keep that split: new logic goes in a Python
module with explicit rules; the model's job is to propose inputs and explain outputs.

## Map

| Path | What it is | Touch it when |
|---|---|---|
| `world.html` | The showcase: Three.js world, five regions, exhibit pages, guided tour. Single file, no build step. | Changing the public experience |
| `assets/*.glb`, `assets/materials/` | 3D bodies for the five nodes and the eye; CC0 textures (credits in `assets/materials/CREDITS.txt`) | Replacing a body or material |
| `assets/demo-*.json` | Example data the exhibits render from. Shapes match the engine's real outputs. | Adding an exhibit |
| `assets/preview/*.png` | Pre-rendered node previews used by the world | Regenerating previews |
| `engine.html` | The engine-room exhibit as a standalone page | Explaining the pipeline |
| `zola.html` | The working shell: eye-and-orbit homepage, five domain nodes, live polling | Changing the product UI |
| `zola-server.py` | Local server on `127.0.0.1:3737`. Serves files and computes `/api/*` routes. | Adding an API |
| `engine/zola_cron.py` | Tick-based deterministic scheduler. Zero LLM calls at runtime. | Scheduling anything |
| `engine/zola_learn.py`, `zola_skills.py`, `zola_capture.py`, `zola_pipeline.py` | The self-improving loop: real outcomes strengthen or decay learned "moves" | Changing how outcomes feed back |
| `engine/zola_verify.py` | Shared verification kernel: every number carries its reference and error band | Any new computed number |
| `engine/zola_survivorship.py`, `zola_signaling.py` | Reasoning filters (Wilson lower bounds; workplace signaling equilibria) | New reasoning rules |
| `engine/zola_screen.py`, `zola_fin_confirm.py`, `zola_itshock.py` | Finance engines. They inform; they never trade. | Market logic |
| `engine/zola_ladder.py`, `zola_sessions.py`, `zola_sweeps.py`, `zola_moves.py` | Career, creative-practice, intel and move engines | Domain logic |
| `engine/build-feeds.sh` | Builds the demo feeds the showcase reads | Refreshing exhibit data |
| `docs/DESIGN-BRIEF.md` | The design law: one language, colour table, type, motion inventory | Any visual change |
| `docs/RESEARCH.md` | Why the design decisions were made | Before arguing with the brief |
| `docs/ROADMAP.md` | Ordered backlog for the showcase | Picking what to build next |
| `docs/2026-10-26-exhibits-brief.md` | Spec for the six exhibit pages | Changing an exhibit |
| `docs/hero/` | Stills of the world, captured with the `?still` harness | README images |

## API routes (zola-server.py)

`/api/quotes`, `/api/history`, `/api/news` (market), `/api/reason` and `/api/ask`
(the lens: classify, recall, judge, verdict), `/api/run` and `/api/confirm` (run an
engine, confirm a move), `/api/control`, `/api/loop` (the learning loop state),
`/api/sky`, `/api/daytracker`. Each handler is a few lines in `do_GET`; read it rather
than this list when adding one.

## Run it

```bash
# the showcase (static, nothing fetched from the network)
python3 -m http.server 8790
# open http://localhost:8790/world.html

# the working shell + engine
python3 zola-server.py            # http://127.0.0.1:3737/zola.html
python3 engine/zola_cron.py list  # the scheduler table
```

Deterministic stills for review or README images:
`world.html?still&region=finance|career|creative|intel|engine` freezes every animation
and transition. Capture with headless Chrome at 1600x900.

## Rules for changes

1. **No personal data, ever.** Do not add JSON with real names, money, employers or
   colleagues. If a feature needs data, add it as `*.local.json` and read it at boot.
2. **No build step.** `world.html` and `zola.html` stay single files served statically.
   Vendored libraries live in `vendor/` and `jsm/`; do not add a bundler.
3. **No network at runtime for the showcase.** Fonts are the only external fetch.
4. **Design law is in `docs/DESIGN-BRIEF.md`.** Flat UI, deep scene. Node hues are
   state, used only as core light, a 2px rule, and sparkline strokes. One shared clock
   for all idle motion. Measured WCAG AA on every text pair.
5. **Deterministic over clever.** A rule in Python beats a prompt. If you add a verdict,
   it must be reproducible from the inputs without a model.
6. **Verify with the harness.** Before and after any visual change, capture the same
   `?still&region=` frames and compare. Lint JS with `node --check` on the extracted
   module.
7. **Comments on their own line.** The world file is parsed by a small extractor; an
   inline `//` after code breaks it.

## Good first extensions

- A sixth region page from a new `assets/demo-*.json`, following the exhibits brief.
- A new reasoning filter in `engine/`, with its own test file and a `/api/reason` hook.
- A mobile layout pass for the exhibit pages (roadmap item 15).
- A quality switch for the scene (roadmap item 14).

## Provenance

Designed and built solo by the owner, AI-assisted end to end (Claude Code daily),
between mid-2026 and now. The private system behind this cut is larger; the parts
that hold personal data, and the tooling built for private use, stay private.
