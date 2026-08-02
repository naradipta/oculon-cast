# Roadmap

Last updated: 2026-08-02

---

## Component status

| Component | Location | Status | Notes |
|---|---|---|---|
| Grounding client | `legacy/visual_grounding_client.py` | Prototype | `FrameGeometry`, letterboxing, forced tool use, crop verification. **Port, don't rewrite.** |
| Element registry | `legacy/element_registry.py` | Prototype | Offline compiler, cached-text grounding, image escalation |
| Registry capture (web) | `legacy/registry_capture.py` | Prototype | Playwright crawler, selector heuristic, manifest emission |
| CLI + run state machine | `oculoncast/cli.py`, `runstate.py` | Not started | Blocks everything else |
| Segmentation | `oculoncast/segment/` | Not started | **Next build.** No AI, no dependencies. |
| Widget discovery + Gate A | `oculoncast/discover/` | Not started | Includes standalone `widgets.html` |
| Screen classifier | `oculoncast/ground/` | Not started | Input format ready (`screens_manifest.json`) |
| Assertions | `oculoncast/assertions/` | Not started | Closed vocabulary + NL parsing |
| Generator | `oculoncast/generate/` | Not started | Target language undecided — see Open Decisions |
| Eval harness | `eval/` | Not started | Gates all optimization work |
| Registry capture (mobile) | — | Backlog | Appium `getPageSource()` XML → bounds + accessibility IDs |
| Web UI (Phase 2) | `web/` | Backlog | Trigger: non-technical QA users |

---

## Build order

1. **CLI skeleton + run state machine.** Every other stage plugs into
   it. `ocast run`, `ocast resume`, `ocast status`, run folder layout,
   `run.json` read/write, cost logging.
2. **Segmentation.** No AI, no upstream dependencies, and the stage
   where the original plan diverged most from what production needs
   (windowing, not frame dedup). Produces `windows.json`.
3. **Widget discovery + Gate A.** Includes the standalone
   `widgets.html` labeling page. This is what unlocks uninstrumented
   recording sources.
4. **Port grounding + registry prototypes** into `ground/` and
   `registry/`, wired to the state machine. Add the screen classifier.
5. **Gate B review page** (`actions.html`), worst-confidence-first.
6. **Assertions** (auto-transition + Gate C NL parsing).
7. **Generator** for the chosen target language.
8. **Eval harness.**

---

## Deferred: token optimization

**Do not implement until baseline grounding accuracy is measured.**
Each of these trades some accuracy for cost, and there is currently no
number to trade against.

| # | Stage | Optimization | Est. saving | Risk |
|---|---|---|---|---|
| 1 | Ground | Send 1 full before-frame + **diff-region crop** instead of 2 full frames (`diff_region` already recorded in `windows.json`) | ~40% of grounding | Low |
| 2 | Classify | **Perceptual hash fast-path** (pHash/dHash), Claude only on ambiguity | ~85% of classify | Low |
| 3 | Ground | **Haiku-first cascade**, escalate to Sonnet below 0.85 | ~30-40% of grounding | Medium |
| 4 | Classify | **State-machine locality** — skip classify unless change magnitude is navigation-sized | ~50% of remaining | Low |
| 5 | Ground | Grounding cache keyed on `(screen_id, diff_region_hash)` | High on repeat flows | Low |
| 6 | Discover | pHash-dedupe screens before widget discovery | ~60% of discovery | Low |
| 7 | Ground | Reduce canvas 1280×800 → 960×600 | ~44% of image tokens | **High** |
| 8 | All | 1-hour cache TTL for long runs | Marginal | Low |

Projected combined effect: **~$0.39 → ~$0.13** per 3-minute video
(≈65% reduction) with 1-4, without any new model dependency.

### On the trained-classifier question

A deep-learning screen classifier was considered and **rejected**:

- Requires labeled training data per app, regenerated every redesign —
  the exact cold-start problem this architecture avoids.
- Breaks the native-multimodal-only constraint and adds model
  artifacts, inference dependencies, and versioning burden.
- Overkill: screens of one app are structurally near-identical. This is
  matching against ~10 known references, not open-world recognition.

**Perceptual hashing achieves the same result** at zero token cost and
zero training, and stays within the DSP exemption alongside ffmpeg.

Item 7 (canvas reduction) is the one to treat as high risk — it
directly degrades small-element discrimination, which is the project's
core success metric.

---

## Open decisions

### 1. First target output: Playwright or Appium?

Affects generator design and which locator field is primary. Both are
recorded in the registry; only the emitter differs.

### 2. Recording source

Instrumented browser vs. external mobile / manual QA recordings.

Gate A (widget labeling) is what makes uninstrumented sources viable,
which resolves the original concern that the video pipeline might be
redundant. But it remains worth confirming: if every target flow can be
recorded via Playwright codegen or trace files, actions are captured
exactly with zero AI inference, and much of stages 2-6 is unnecessary
for those flows.

The video pipeline earns its complexity for: real mobile devices,
manual QA sessions, user-submitted bug reports, third-party apps,
legacy recording archives.

### 3. Trademark

"Oculon" is in use by several unrelated software companies (market
intelligence, financial planning, cybersecurity). None in UI test
automation, and the compound "Oculon Cast" is distinctive enough for
internal or open-source use. A proper class-42 search is required
before any commercial release.

---

## Success metric

**Element-ID grounding accuracy**, measured by the eval harness.

Everything downstream depends on it — screen classification quality,
assertion resolution, and generated-script reliability all sit on top
of correctly identifying which element was acted upon.

Method: one short recording of a known flow, 10-15 hand-labeled
keyframe pairs, run through the pipeline, compare against ground truth.
Gate B corrections from real runs extend the dataset for free.

Until this number exists, all cost and accuracy claims in this
repository are **designed reasoning, not measurement**.
