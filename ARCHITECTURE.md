# Architecture

## Premise

Convert screen recordings into executable UI automation scripts using
**native multimodal AI only**. All semantic visual work — what is on
screen, what the user did, which element they touched — is performed by
Claude vision. No OpenCV, YOLO, or external OCR.

`ffmpeg` and `Pillow` are permitted for keyframe extraction, resizing,
cropping, and hashing. These are signal processing and image I/O, not
computer vision.

### Why this constraint is worth keeping

Classical CV pipelines require per-app template libraries, break on
theme changes, and need retraining or retuning for every redesign. An
LMM matches on *shape, position, role, and text semantics*, which is
what makes dynamic elements (changing badge counts, per-user avatars,
rotating banners) tractable at all. The constraint is the product.

---

## Control flow: Python orchestrates, AI answers

Claude is **not** the orchestrator. A deterministic Python pipeline
drives every run and calls the model at four defined points.

| | AI orchestrates | Python orchestrates (chosen) |
|---|---|---|
| Determinism | Same video → different call sequences | Same video → same pipeline |
| Failure handling | Model decides retries | Explicit retry, backoff, resume |
| Cost | Unbounded agentic loop | Predictable: N windows × known calls |
| Debuggability | Reasoning trace | Per-stage artifacts on disk |

Claude-as-orchestrator remains useful for *interactive* debugging
("why did step 14 fail?"), but never for the production path.

---

## Pipeline

```
┌────────────────── OFFLINE (once per app version) ──────────────────┐
│  Source A: Playwright crawler   → elements_manifest.json           │
│  Source B: Widget discovery + Gate A labeling → same format        │
│                    ↓                                               │
│  Registry compiler (Claude describes each element once)            │
│                    → registry.json  (text descriptions, no images) │
└────────────────────────────────────────────────────────────────────┘

┌────────────────── PER RUN ─────────────────────────────────────────┐
│ 1. UPLOAD          video → runs/{run_id}/input/          [no AI]   │
│ 2. SEGMENT         ffmpeg → change signal → windows      [no AI]   │
│ 3. DISCOVER        widget candidates (if registry gaps)  [Sonnet]  │
│    ⏸ GATE A       user labels widgets                             │
│ 4. CLASSIFY        screen per window                     [Haiku]   │
│ 5. GROUND          frame pair + cached registry          [Sonnet]  │
│    5b. ESCALATE    conf < 0.75 → 2-3 candidate crops     [Sonnet]  │
│ 6. ACTION LOG      actions.json                          [no AI]   │
│    ⏸ GATE B       user reviews / edits steps                      │
│ 7. ASSERTIONS      transitions + NL input                [Sonnet]  │
│    ⏸ GATE C       user writes validations                         │
│ 8. GENERATE        script emission                       [no AI]   │
│ 9. REPORT          confidence, flags, cost               [no AI]   │
└────────────────────────────────────────────────────────────────────┘
```

Five of nine stages involve no AI at all. That ratio is intentional and
should be defended.

---

## Stage detail

### 2. Segmentation — windowing, not dedup

The most important correction in the design.

Naive "remove similar frames" fails on three real cases:

- **Continuous gestures.** A scroll produces 20+ consecutively
  different frames representing **one** action. Frame-level dedup keeps
  all 20 and emits 20 fake scroll steps.
- **Delayed response.** A tap whose visual effect arrives 800ms later
  (network round trip) — the changed frame pair no longer brackets the
  tap, so the action is attributed to the wrong moment.
- **No visual change.** A disabled button, a rejected validation. Dedup
  deletes the frame outright and the action vanishes from the script.

Correct algorithm:

```
frames → per-frame change signal (ffmpeg scene filter / frame delta)
       → cluster consecutive changed frames into ACTION WINDOWS
       → per window emit (frame_before_window, frame_after_settle)
       → classify window: discrete (tap/type) vs continuous (scroll/swipe)
```

One window = one candidate action = one grounding call. Continuous
windows collapse to a single scroll action with a direction, never
per-frame deltas.

### 3. Widget discovery + Gate A

Runs only when the registry lacks coverage for screens seen in the
video. This is what makes **uninstrumented recording sources** viable —
real mobile devices, manual QA sessions, user bug reports, third-party
apps.

```
1. Sample distinct screens from the segmented video
2. Claude enumerates interactive candidates per screen
   → bbox, role, visible text, proposed element_id, dynamic? guess
3. Crop each candidate from the frame (Pillow)
4. Emit gate_a/widgets.html — crops embedded as base64, no server
5. USER: confirm/rename element_id, fix role, tick dynamic_content,
   paste real locator if known
6. Import → elements_manifest.json → compile_registry()
```

Design principles:

- **AI proposes, human certifies.** The user edits defaults rather than
  authoring from scratch — ~10 minutes for 40 widgets, not an hour.
- **Show the crop next to the row.** A plain CSV cannot, which is why
  the HTML page exists. CSV remains available for bulk edits.
- **Empty `locator` is legitimate.** For real-device recordings the
  user often has no selector. The generator then falls back to
  text/coordinate matching and flags the step. Do not fabricate
  selectors.
- **Incremental.** Only unknown widgets appear. A second video of the
  same app skips Gate A entirely.

### 4. Screen classification

One frame against a short list of known screens — an easy,
high-frequency task, so it routes to Haiku. Its output selects the
registry subset (~15 elements instead of ~200) for the grounding call.

Must support **screen + overlay** states. Modals and toasts are not
screens; treating them as such tanks apparent accuracy.

Screen *transitions* are also the highest-value free signal in the
pipeline: `cart → checkout` after step 12 auto-proposes a navigation
assertion.

### 5. Grounding

The core call. Inputs: the before/after frame pair (letterboxed to
1280×800) plus the screen's registry subset as **cached text** in the
system prompt. Output via forced tool use: action type, `element_id`,
coordinates, confidence, runner-up candidates.

**Why text descriptions instead of reference images at runtime.** An
LMM does not need reference pixels to recognise "pill-shaped orange
button labelled Checkout, bottom-right" — the description *is* the
template. ~50 text tokens per element versus hundreds of visual tokens
per crop, and unlike images the text is a stable prefix that prompt
caching amortises across every call in the video.

**Coordinate determinism.** Frames are letterboxed client-side with
uniform scaling; `FrameGeometry` records scale and padding so returned
coordinates invert exactly to native device pixels. Skipping
client-side resizing lets the API downscale silently and every
coordinate drifts.

**Dynamic elements** are matched on invariants — container shape,
normalised `expected_region` plus tolerance, and role — never on pixel
content.

### 5b. Escalation

Fires only below 0.75 confidence or on `UNKNOWN`. Attaches the actual
reference crops of the **top 2-3 candidates only**. This is the sole
runtime path that spends visual tokens on reference images.

Sending the whole image dataset every call is both expensive *and* less
accurate — more candidates in context means more confusion between
visually similar elements.

### 6. Action log — the intermediate representation

`actions.json` is a durable, reviewable, editable artifact between
understanding and code generation.

Why it must exist: grounding is approximate by design. A human can
correct the log in minutes, and regenerating scripts from a corrected
log is free. Regenerating from video after every fix is not. It is also
the only place low-confidence steps can be flagged before they become
silent bugs in a test suite.

Gate B corrections double as the evaluation dataset — every user edit
is a labelled ground-truth disagreement.

### 7. Assertions

A recording yields **actions**. A *test* needs **assertions**, or the
output is a click macro that passes while the app is broken.

Two sources:

- **Auto-proposed from screen transitions** — free, and usually the
  most valuable assertions in the script.
- **Natural language from the user** (Gate C), parsed into a
  **closed assertion vocabulary**: `visible`, `not_visible`,
  `text_equals`, `text_contains`, `url_matches`, `count_equals`,
  `enabled`.

Free-form assertion code generation produces uncompilable output; a
constrained enum plus a tool schema always produces valid results. When
a widget reference cannot be resolved against the registry, return a
clarification request listing nearby candidates — never invent an
`element_id`.

### 8. Generation

Deterministic emission from `actions.json` + assertions.

Rules:

- **Semantic locators first.** Coordinates only when no handle exists,
  and always flagged `needs_review` in the output.
- **Scroll-until-visible**, never pixel deltas — magnitude inferred
  from keyframes is approximate.
- **Masked input → `{{PARAM}}` placeholders.** Passwords and
  autocorrected text are partially unrecoverable from frames.
  Parameterised tests are better practice regardless.

---

## Interface: CLI-first, files as gates

The three gates pause the pipeline and wait for human input. A web app
makes that pretty; a run folder makes it work — the folder **is** the
state machine, so runs are resumable, diffable, and git-committable.

The one place pure CLI fails is widget labeling: you cannot label 40 UI
elements from a CSV without seeing them. Solved with a standalone HTML
file, crops embedded as base64 — no server, no framework.

Phase 2 wraps the same core functions in FastAPI + Next.js. Gates
become web forms; the run folder becomes object storage plus a `runs`
table. Nothing in the pipeline changes.

The trigger for Phase 2 is non-technical users: as long as the users
are engineers, CLI is *better*, not merely faster to build.

---

## Cost model

Rates (per million tokens): Sonnet 4.6 $3 / $15, Haiku 4.5 $1 / $5.
Prompt caching cuts cached input by 90%; batch processing 50%.

A 1280×800 frame ≈ 1,365 input tokens.

| Call | Model | Cost / call |
|---|---|---|
| Screen classification | Haiku 4.5 | ~$0.002 |
| Action grounding | Sonnet 4.6 | ~$0.011 |
| Escalation (~15% of steps) | Sonnet 4.6 | ~$0.009 |
| Widget description (offline) | Sonnet 4.6 | ~$0.003 |

**≈ $0.40 per 3-minute recording** (~28-30 action windows).

Action-window segmentation is the reason this is affordable: ~30 calls
rather than ~80 raw keyframe pairs.

---

## Known limitations

1. **Same-model verification.** The crop verification pass uses the
   same model that grounded — correlated errors. Accepted for v1.
2. **Registry drift.** App redesigns silently stale the registry.
   Monitor `UNKNOWN` and low-confidence rates; a spike means
   re-capture.
3. **Grounding is approximate by design.** Anthropic documents
   coordinate output as approximate. The verification loop and the
   semantic-locator preference exist to absorb this.
4. **Information lost in keyframes is unrecoverable.** Masked typed
   text, sub-frame tap ripples, imprecise scroll magnitude. Handled by
   convention at generation time, not recovered.
5. **No measured baseline accuracy.** Everything above is designed
   reasoning, not measurement. The eval harness is the gate on all
   optimization work.
