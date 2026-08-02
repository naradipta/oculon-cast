# Oculon Cast

**Turn screen recordings into Playwright and Appium tests, using vision alone.**

Oculon Cast takes a screencast of someone using an app and produces an
executable UI automation test script. No instrumentation of the recorded
session is required — which means it works on real mobile devices,
manual QA sessions, and user-submitted bug report videos.

*oculus* (eye) + *cast* (screencast).

---

## How it works

```
screencast ──► action windows ──► screen + element identification ──► action log ──► test script
                  [ffmpeg]              [Claude vision]              [reviewable]    [Playwright/Appium]
```

The core constraint: **all visual understanding is done by native
multimodal AI** (Claude vision via the Anthropic API). No OpenCV, no
YOLO, no OCR libraries. `ffmpeg` and `Pillow` handle frame extraction
and image I/O only.

Three human checkpoints keep output trustworthy:

| Gate | You do | Why |
|---|---|---|
| **A** Widget labeling | Confirm AI-proposed element names + locators | Once per app version. Skipped when the registry already covers the screens. |
| **B** Action review | Fix low-confidence steps | Grounding is approximate by design; 2 minutes here beats debugging a flaky suite. |
| **C** Validations | Describe assertions in plain language | A recording gives you clicks. A *test* needs assertions. |

---

## Install

```bash
git clone <repo> oculon-cast
cd oculon-cast
python -m venv .venv && source .venv/bin/activate
pip install -e .
playwright install chromium          # only for web registry capture

export ANTHROPIC_API_KEY=sk-ant-...
```

Requires Python 3.11+ and `ffmpeg` on PATH.

---

## Quick start

```bash
$ ocast run recordings/checkout.mp4 --app myshop --platform web

  ✓ Segmented 1,847 frames → 28 action windows
  ✓ Registry check: 12 known widgets, 9 unknown
  ⏸ GATE A — 9 widgets need labeling
    → open runs/2026-08-02_143022/gate_a/widgets.html
    → save widgets.labeled.json back into gate_a/
    → then: ocast resume 2026-08-02_143022
```

Open `widgets.html` in any browser — element crops are embedded, no
server needed. Confirm the proposed names, tick anything whose content
changes at runtime (badges, avatars), paste real selectors if you have
them. Download the JSON, drop it back, and resume.

```bash
$ ocast resume 2026-08-02_143022

  ✓ Registry compiled (21 widgets)
  ✓ Classified 28 windows   ✓ Grounded 28 actions
    24 high-confidence · 4 flagged for review
  ⏸ GATE B — review actions
```

`gate_b/actions.html` shows before/after frames per step, sorted
worst-confidence-first. Fix the four flagged steps (or edit
`actions.json` directly), resume.

```bash
$ ocast resume 2026-08-02_143022

  ✓ Auto-proposed 6 assertions from screen transitions
  ⏸ GATE C — add validations in plain language
```

Edit `gate_c/validations.txt`:

```
@step 12  after checkout, total should show 150000
@step 12  success banner must appear
@step 20  cart badge should be empty
```

Resume once more:

```bash
$ ocast resume 2026-08-02_143022

  ✓ Parsed 3 custom validations
  ✓ Generated output/checkout_flow.spec.ts (28 steps, 9 assertions)
  ✓ Cost: $0.41 · Report: output/report.md
```

---

## Commands

| Command | Purpose |
|---|---|
| `ocast run <video> --app <name> --platform web\|mobile` | Start a new run |
| `ocast resume <run_id>` | Continue after a gate |
| `ocast status <run_id>` | Current stage + pending gate |
| `ocast list` | All runs and their stages |
| `ocast registry show --app <name>` | Inspect the compiled registry |
| `ocast rerun <run_id> --from <stage>` | Re-run one stage against cached artifacts |

`--from` accepts `segment`, `discover`, `ground`, `assert`, `generate`.
Use it when tuning prompts — no re-upload, no re-labeling.

---

## Run folder

Everything a run produces lives on disk. The folder **is** the state
machine, which makes runs resumable, diffable, and inspectable.

```
runs/2026-08-02_143022_checkout-flow/
├── run.json                  state, timestamps, token + cost log
├── input/checkout.mp4
├── frames/
├── windows.json              action windows from segmentation
├── gate_a/
│   ├── widgets.html          ← open in browser to label
│   ├── widgets.csv           bulk-edit alternative
│   ├── crops/
│   └── widgets.labeled.json  ← drop back here
├── registry.json             compiled; reused across runs
├── gate_b/
│   ├── actions.json          ← editable
│   └── actions.html          visual review
├── gate_c/validations.txt    ← plain language
└── output/
    ├── checkout_flow.spec.ts
    └── report.md
```

---

## The registry

Element identification is grounded against a **registry** — a per-app
catalogue of UI widgets. Two ways to build it:

1. **Crawler** (`ocast registry capture`) — a Playwright crawler walks
   declared screens and extracts elements, crops, regions, and
   selectors automatically. Best when you control the app.
2. **Widget labeling** (Gate A) — Claude proposes widget candidates
   from the video itself; you confirm them in the browser. This is what
   makes uninstrumented sources (real devices, QA sessions, bug report
   videos) work at all.

Both produce the same manifest format. See `docs/SCHEMAS.md`.

---

## Cost

Roughly **$0.40 per 3-minute recording** (~28 action windows), using
Haiku for screen classification, Sonnet for grounding, and prompt
caching on the registry. Every run reports its actual cost.

Widget labeling and registry compilation are one-time per app version,
not per run.

---

## Documentation

- [`CLAUDE.md`](CLAUDE.md) — operating guide for AI coding sessions
- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — pipeline detail and design rationale
- [`docs/SCHEMAS.md`](docs/SCHEMAS.md) — every JSON contract
- [`docs/ROADMAP.md`](docs/ROADMAP.md) — status, deferred work, open decisions

---

## Status

Early development. Grounding, registry compilation, and web registry
capture are built as prototypes; the CLI, segmentation, discovery,
assertions, and generator stages are in progress. See
[`docs/ROADMAP.md`](docs/ROADMAP.md).

**No baseline grounding accuracy has been measured yet.** The
evaluation harness is the next priority and gates all cost/accuracy
optimization work.
