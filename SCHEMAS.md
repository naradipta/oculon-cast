# Schemas

Every JSON contract that crosses a stage boundary. Stages may not
mutate a previous stage's artifact — always write a new file.

---

## `run.json` — state machine

```json
{
  "run_id": "2026-08-02_143022_checkout-flow",
  "app": "myshop",
  "platform": "web",
  "input_video": "input/checkout.mp4",
  "stage": "awaiting_review",
  "created_at": "2026-08-02T14:30:22Z",
  "updated_at": "2026-08-02T14:41:07Z",
  "stages_completed": ["upload", "segment", "discover", "gate_a",
                       "classify", "ground"],
  "cost": {
    "total_usd": 0.41,
    "by_stage": { "discover": 0.06, "classify": 0.05, "ground": 0.30 },
    "tokens": { "input": 128400, "cached_input": 91200, "output": 6180 }
  },
  "counts": { "frames": 1847, "windows": 28, "flagged": 4 }
}
```

**Stage values:**
`uploaded` → `segmented` → `discovering` → `awaiting_labels` →
`classified` → `grounded` → `awaiting_review` → `awaiting_validation` →
`generating` → `done` (plus `failed`).

---

## `windows.json` — segmentation output

```json
{
  "fps": 10,
  "total_frames": 1847,
  "windows": [
    {
      "index": 0,
      "frame_range": [412, 447],
      "type": "discrete",
      "frame_before": "frames/000411.png",
      "frame_after": "frames/000448.png",
      "change_magnitude": 0.34,
      "diff_region": [0.61, 0.86, 0.99, 0.98]
    },
    {
      "index": 1,
      "frame_range": [520, 604],
      "type": "continuous",
      "frame_before": "frames/000519.png",
      "frame_after": "frames/000605.png",
      "direction": "down",
      "change_magnitude": 0.71
    }
  ]
}
```

`type` is `discrete` (tap, type, long-press) or `continuous` (scroll,
swipe). `diff_region` is normalized `[x0, y0, x1, y1]` — reserved for
the deferred diff-region cropping optimization.

---

## `elements_manifest.json` — human-owned ground truth

Produced by **either** the Playwright crawler **or** Gate A labeling.
Same format from both sources.

```json
[
  {
    "element_id": "btn_checkout",
    "screen": "cart",
    "reference_image": "gate_a/crops/btn_checkout.png",
    "expected_region": [0.62, 0.88, 0.98, 0.97],
    "region_tolerance": 0.05,
    "dynamic_content": false,
    "locator": {
      "playwright": "getByTestId('checkout-btn')",
      "appium": "accessibility_id=checkout_btn"
    },
    "selector_strategy": "testid",
    "needs_review": false
  },
  {
    "element_id": "badge_cart_count",
    "screen": "home",
    "reference_image": "gate_a/crops/badge_cart_count.png",
    "expected_region": [0.90, 0.02, 0.97, 0.07],
    "region_tolerance": 0.03,
    "dynamic_content": true,
    "locator": { "playwright": null, "appium": null },
    "selector_strategy": "none",
    "needs_review": true
  }
]
```

**Field ownership:**

| Field | Owner | Note |
|---|---|---|
| `element_id` | Human | Must be human-chosen or it changes every recompile |
| `screen` | Human / crawler | Drives runtime registry subsetting |
| `reference_image` | Crawler / discovery | Crop path |
| `expected_region` | Crawler / discovery | Normalized to the 1280×800 canvas |
| `dynamic_content` | Human (overrides AI guess) | Only the human knows an avatar varies per user |
| `locator` | Human / crawler | May be `null` for uninstrumented sources |
| `selector_strategy` | Crawler | `testid` \| `stable_id` \| `role_name` \| `text` \| `name_attr` \| `css_fallback` \| `none` |
| `needs_review` | Derived | True when strategy is `text` or weaker |

---

## `registry.json` — compiled

Manifest fields **plus** Claude-generated description fields. Produced
once per app version by `compile_registry()`.

```json
[
  {
    "element_id": "badge_cart_count",
    "screen": "home",
    "expected_region": [0.90, 0.02, 0.97, 0.07],
    "region_tolerance": 0.03,
    "dynamic_content": true,
    "locator": { "playwright": "locator('[data-testid=cart-badge]')" },
    "reference_image": "gate_a/crops/badge_cart_count.png",

    "description": "Small red circular badge with white numeric text, overlaid on the top-right corner of the cart icon in the header.",
    "role": "badge",
    "shape_invariants": "Red circle, ~3% of screen width, pinned to the cart icon corner; only the digit inside changes."
  }
]
```

`reference_image` is retained for the escalation path only — it is
never sent during normal grounding.

---

## `screens_manifest.json`

```json
[
  {
    "screen": "cart",
    "url": "https://app.example.com/cart",
    "reference_image": "capture/screens/cart.png",
    "description": "Shopping cart page: line-item list on the left, order summary panel on the right, checkout button bottom-right.",
    "overlays": ["modal_remove_confirm", "toast_item_removed"]
  }
]
```

`overlays` exists because modals and toasts are **not** screens.
Classification returns `screen` plus optional `overlay`.

---

## `actions.json` — the intermediate representation

The reviewable, editable contract between understanding and codegen.

```json
{
  "run_id": "2026-08-02_143022_checkout-flow",
  "app": "myshop",
  "steps": [
    {
      "step": 12,
      "window": 11,
      "frame_range": [412, 447],
      "screen": "cart",
      "overlay": null,
      "action": "tap",
      "element_id": "btn_checkout",
      "confidence": 0.91,
      "native_center": [512, 1180],
      "locator": {
        "playwright": "getByTestId('checkout-btn')",
        "appium": "accessibility_id=checkout_btn"
      },
      "typed_text": null,
      "escalated": false,
      "needs_review": false,
      "reasoning": "Button pressed state visible in before-frame; checkout screen renders in after-frame.",
      "edited_by_human": false
    },
    {
      "step": 13,
      "window": 12,
      "frame_range": [520, 604],
      "screen": "checkout",
      "action": "scroll",
      "element_id": null,
      "direction": "down",
      "confidence": 0.88,
      "needs_review": false,
      "edited_by_human": false
    },
    {
      "step": 14,
      "window": 13,
      "screen": "checkout",
      "action": "type",
      "element_id": "txt_card_number",
      "typed_text": "{{CARD_NUMBER}}",
      "masked": true,
      "confidence": 0.62,
      "escalated": true,
      "needs_review": true,
      "edited_by_human": false
    }
  ]
}
```

**Action values:** `tap`, `double_tap`, `long_press`, `type`, `swipe`,
`scroll`, `none`.

`edited_by_human` is set by Gate B and must be preserved — those
records are the evaluation dataset.

---

## `validations.txt` — Gate C input

```
# One validation per line.
# Format:  @step <n>  <plain language>

@step 12  after checkout, total should show 150000
@step 12  success banner must appear
@step 20  cart badge should be empty
```

---

## `assertions.json` — parsed output

```json
[
  {
    "after_step": 12,
    "source": "auto_transition",
    "assertions": [
      { "type": "url_matches", "expected": "/checkout" }
    ]
  },
  {
    "after_step": 12,
    "source": "user_nl",
    "raw": "after checkout, total should show 150000",
    "assertions": [
      { "type": "text_contains", "element_id": "lbl_total", "expected": "150000" },
      { "type": "visible", "element_id": "banner_success" }
    ]
  },
  {
    "after_step": 20,
    "source": "user_nl",
    "raw": "cart badge should be empty",
    "status": "needs_clarification",
    "message": "No registry widget matches 'cart badge' on screen 'home'.",
    "candidates": ["badge_cart_count", "icon_cart"]
  }
]
```

**Closed assertion vocabulary** — do not extend without updating the
generator: `visible`, `not_visible`, `text_equals`, `text_contains`,
`url_matches`, `count_equals`, `enabled`.

Unresolvable references return `needs_clarification` with candidates.
Never invent an `element_id`.

---

## Tool schemas (Anthropic API)

All Claude calls use forced tool use:
`tool_choice={"type": "tool", "name": "..."}`.

| Stage | Tool name |
|---|---|
| Widget description (offline) | `describe_ui_element` |
| Widget discovery | `enumerate_ui_elements` |
| Screen classification | `classify_screen` |
| Grounding | `report_matched_action` |
| Escalation | `report_matched_action` (reused) |
| Assertion parsing | `parse_validations` |

Always assert the `tool_use` block exists; check `stop_reason` when it
does not.
