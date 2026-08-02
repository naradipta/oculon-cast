# CLAUDE.md

Operating guide for AI coding sessions in this repository.
Read this before writing code. If a change contradicts anything here,
stop and raise it rather than silently deviating.

---

## What this project is

**Oculon Cast** turns screen recordings into executable UI automation
tests (Playwright / Appium), using vision alone.

`screencast → action log → test script`

The name: *oculus* (eye) + *cast* (screencast). Repo `oculon-cast`,
Python package `oculoncast`, CLI `ocast`.

---

## Non-negotiable constraints

These are architectural commitments, not preferences. Violating them
invalidates the project's premise.

| Rule | Detail |
|---|---|
| **Native multimodal AI only** | All *semantic* visual work goes through the Anthropic API. **No OpenCV, no YOLO, no Tesseract, no external OCR, no trained CV models.** |
| **Signal processing is allowed** | `ffmpeg` (keyframe extraction, scene-change detection) and `Pillow` (resize, crop, letterbox, hash) are I/O and DSP, not computer vision. Permitted. |
| **Never send video to the API** | The Messages API accepts images only. Keyframes are extracted client-side. |
| **One coordinate space** | Everything is 1280x800. Capture viewport = grounding canvas. Never change this without updating `FrameGeometry` and re-running the eval harness. |
| **Semantic locators over coordinates** | Generated scripts emit `getByTestId(...)` / accessibility IDs. Raw coordinates appear only when no semantic handle exists, and must be flagged `needs_review`. |
| **Deterministic Python orchestrates; AI is a service** | Python drives the pipeline. Claude is called at 4 defined points. Never build an agentic loop that lets the model decide the pipeline sequence. |

---

## Pipeline (9 stages, 3 human gates)

```
1. UPLOAD           video → runs/{run_id}/input/            [no AI]
2. SEGMENT          ffmpeg → change signal → action windows [no AI]
3. WIDGET DISCOVERY only if registry incomplete             [Claude]
   GATE A           user labels widgets → registry
4. CLASSIFY         which screen per window                 [Haiku]
5. GROUND           frame pair + cached registry → element  [Sonnet]
   5b. ESCALATE     confidence < 0.75 → candidate crops     [Sonnet]
6. ACTION LOG       actions.json, sorted by confidence      [no AI]
   GATE B           user reviews / edits steps
7. ASSERTIONS       auto from transitions + NL input        [Sonnet]
   GATE C           user writes plain-language validations
8. GENERATE         actions + assertions → script           [no AI]
9. REPORT           confidence summary, flags, cost         [no AI]
```

Stages 1, 2, 6, 8, 9 contain **zero AI calls**. Keep it that way — if
you find yourself adding a Claude call to one of them, the design is
drifting.

Full detail: `docs/ARCHITECTURE.md`.

---

## Critical implementation rules

### Segmentation is windowing, NOT frame dedup

The single most common mistake. Do **not** "remove similar frames."

- A scroll produces 20+ different frames = **one** action. Naive dedup
  emits 20 fake scroll steps.
- A tap with a delayed network response changes frames 800ms later.
  The changed pair no longer brackets the tap.
- A tap with no visual change (disabled button) has **no** frame diff
  and gets deleted entirely by dedup.

Correct approach: compute a per-frame change signal, cluster
consecutive changed frames into **action windows**, then emit
`(frame_before_window, frame_after_settle)` per window. Classify each
window as discrete (tap/type) or continuous (scroll/swipe). One window
= one action = one Claude call.

### Coordinates must round-trip deterministically

Frames are letterboxed client-side into 1280x800 with uniform scaling
(never stretch). `FrameGeometry` records scale + padding offsets so
Claude's returned coordinates invert exactly to native device pixels.
If you skip client-side resizing, the API downscales silently and every
coordinate drifts.

### Structured output via forced tool use, always

Never prompt for JSON and parse the text. Every Claude call defines a
tool schema and sets
`tool_choice={"type": "tool", "name": "..."}`. Then assert the
`tool_use` block exists and check `stop_reason` on failure.

### Registry text is cached, reference images are not sent

The registry rides in the **system prompt** as text with
`cache_control: {"type": "ephemeral"}`. Reference element images are
sent **only** in the escalation path (2-3 candidates, below 0.75
confidence). Never attach the full image dataset per frame — it blows
up cost and *degrades* accuracy.

### The run folder IS the state machine

Every stage writes artifacts to `runs/{run_id}/`. `run.json` holds the
current stage. Gates pause the pipeline; `ocast resume` picks up from
disk. No in-memory pipeline state may be required to continue a run.

### Model routing

| Task | Model | Why |
|---|---|---|
| Screen classification | `claude-haiku-4-5` | One image vs. short list; 3x cheaper |
| Widget description (offline) | `claude-sonnet-4-6` | Quality matters, runs once |
| Action grounding | `claude-sonnet-4-6` | Accuracy determines everything downstream |
| Escalation / disambiguation | `claude-sonnet-4-6` | Hardest cases |
| NL → assertions | `claude-sonnet-4-6` | Constrained vocabulary, needs precision |

Do **not** use `claude-3-5-sonnet-*` — deprecated generation.
Model IDs live in `oculoncast/config.py`. Never hardcode elsewhere.

---

## Repo layout

```
oculon-cast/
├── CLAUDE.md                  ← this file
├── README.md
├── docs/
│   ├── ARCHITECTURE.md        pipeline detail, design decisions
│   ├── SCHEMAS.md             every JSON contract
│   └── ROADMAP.md             status, deferred work, open decisions
├── oculoncast/
│   ├── config.py              model IDs, thresholds, canvas size
│   ├── runstate.py            run folder + state machine
│   ├── segment/               ffmpeg → action windows       [no AI]
│   ├── discover/              widget candidates → gate A    [Claude]
│   ├── registry/              manifest → registry.json
│   ├── ground/                classify + ground + escalate  [Claude]
│   ├── assertions/            NL → assertion objects        [Claude]
│   ├── generate/              actions → script              [no AI]
│   └── cli.py                 ocast entrypoint
├── eval/                      grounding accuracy harness
├── legacy/                    prototypes — see note below
└── runs/                      per-run artifacts (gitignored)
```

### Existing prototype modules

Three modules were built before the CLI structure existed. They are
**correct and should be ported, not rewritten**:

| Prototype | Ports into | Contains |
|---|---|---|
| `visual_grounding_client.py` | `ground/` | `FrameGeometry`, `prepare_frame()`, letterboxing, crop self-verification |
| `element_registry.py` | `registry/` + `ground/` | registry schema, offline compiler, cached-text grounding, image escalation |
| `registry_capture.py` | `discover/` | Playwright crawler, selector priority heuristic, manifest emission |

---

## Conventions

- **Python 3.11+**, type hints everywhere, `from __future__ import annotations`.
- **Dataclasses** for internal structures; **JSON schemas** for anything
  crossing a stage boundary or hitting the API.
- **No stage may mutate a previous stage's artifact.** Write a new file.
- **Every Claude call logs** tokens in/out + cost to `run.json`. Cost
  visibility is a feature, not instrumentation debt.
- **Fail loud on schema mismatch**, retry with backoff on API errors.
  Never silently continue past a failed grounding call — mark the step
  `needs_review` and keep going.
- **Confidence threshold is 0.75** (escalation) and **0.90**
  (auto-collapse in review UI). Both in `config.py`.
- Selector priority order, everywhere:
  `data-testid` → stable `id` → role+name → exact text → CSS fallback.
  Anything at `text` or below is flagged `needs_review`.

---

## CLI contract

```bash
ocast run <video> --app <name> --platform web|mobile
ocast resume <run_id>
ocast status <run_id>
ocast list
ocast registry show --app <name>
ocast rerun <run_id> --from segment|discover|ground|assert|generate
```

`--from` is essential for development: re-run one stage against cached
frames and registry without re-uploading or re-labeling.

---

## Gates: how humans interact

No web server in v1. Gates are files.

- **Gate A — widget labeling.** Generates
  `gate_a/widgets.html`: a **standalone HTML file with element crops
  embedded as base64**, no server, no framework. User labels in the
  browser, downloads `widgets.labeled.json`, drops it back in
  `gate_a/`. CSV export/import exists as a bulk-edit path.
- **Gate B — action review.** `gate_b/actions.json` is directly
  editable. `gate_b/actions.html` renders frames + confidence, sorted
  **worst-confidence-first** (not chronological), auto-collapsing
  anything above 0.90. Insert/delete of steps must be supported.
- **Gate C — validations.** Plain text, one per line:
  ```
  @step 12  after checkout, total should show 150000
  @step 12  success banner must appear
  ```

Gate B edits are also the eval dataset — every user correction is
labeled ground truth. Preserve them.

---

## Things that will bite you

1. **Typed text is partially unrecoverable.** Password fields show
   dots; autocorrect changes what landed. Convention: infer from the
   *after*-frame field content; masked fields become `{{PARAM}}`
   placeholders. Parameterized tests are better practice anyway.
2. **Scroll magnitude is approximate.** Generate `scrollIntoView` /
   `UiScrollable` — never replay pixel deltas.
3. **Same-model verification = correlated errors.** The crop
   verification pass uses the same model that grounded. Known v1
   limitation; document, don't pretend otherwise.
4. **Registry drift.** App redesigns silently stale the registry.
   Monitor `UNKNOWN` + low-confidence rate; a spike means re-capture.
5. **Screens vs. overlays.** Modals and toasts are not screens. The
   classifier must support `screen + overlay` states or accuracy will
   look worse than it is.
6. **Assertions must be a closed vocabulary.** `visible`,
   `not_visible`, `text_equals`, `text_contains`, `url_matches`,
   `count_equals`, `enabled`. Free-form assertion codegen produces
   uncompilable output. If a widget reference can't be resolved,
   **return a clarification request with candidates** — never invent an
   `element_id`.

---

## Before you optimize

**There is currently no measured baseline accuracy.** The eval harness
(`eval/`) is deferred but remains the gate on every cost/accuracy
tradeoff. Do not apply token optimizations (perceptual-hash
classification fast-path, diff-region cropping, Haiku-first grounding
cascade, reduced canvas resolution) until baseline grounding accuracy is
measured. See `docs/ROADMAP.md` § Deferred.

Reducing the grounding canvas below 1280x800 is the one optimization to
treat as **high risk** — it directly degrades small-element
discrimination, which is the project's core success metric.

---

## Open decisions

Do not resolve these unilaterally in code. Raise them.

1. **First target output** — Playwright or Appium? Affects generator
   design.
2. **Recording source** — instrumented browser vs. external mobile /
   manual QA recordings. The widget-labeling gate (Gate A) is what
   makes uninstrumented sources viable; if all sources are
   instrumentable, much of the video pipeline is redundant.
3. **Trademark** — "Oculon" is used by several unrelated software
   companies. Fine for internal/OSS. Requires a proper search before
   any commercial release.
