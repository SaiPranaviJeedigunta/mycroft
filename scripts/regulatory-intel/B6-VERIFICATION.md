# B6 — Alert email silently dropped on exactly 1 high-priority item: verification & fix

**Date:** 2026-10-15 · **Workflow:** `workflow.dev.json` (`Generate Email` node)

## The bug

The real alert-email node (distinct from the "all clear" status email fixed in B5) gates whether
to build the HTML alert at all with:

```js
const hasData = newItems && newItems.length > 1 && newItems[0].json && Object.keys(newItems[0].json).length > 1;

if (!hasData) {
  return [{ json: { has_items: false, item_count: 0, items: [] } }];
}

// If no new high priority items, return empty to skip email
if (newItems.length === 0) {
  return [];
}
```

`newItems.length > 1` requires **strictly more than one** qualifying item before the node will
build an email. A run with exactly **one** new high-priority/critical item fails this check,
falls into the `!hasData` branch, and returns `has_items: false` — which the downstream `If` node
routes to its empty dead-end output (verified: the `If` node's true-output connection array is
`[]`; only the false branch wires to `Send Email Alert`). The result is a single urgent item never
generating an alert email at all.

The `if (newItems.length === 0) { return []; }` check two lines below is dead code even before this
fix — `0 > 1` is already false, so the function always returns on the line above it first. This is
corroborating evidence the author meant `newItems.length > 0` (at least one item) and mistyped `1`.

## Why this matters

This is the opposite failure mode from B5: B5 under-reported what the "no news" email claims to
monitor; B6 means the one scenario this system exists to catch — a single genuinely urgent
filing — can go completely unreported, with no error, no log entry a human would notice, and no
visible symptom besides an alert that quietly never arrives.

## The fix

One-character-class change, `workflow.dev.json`, `Generate Email` node, the `hasData` guard:

```diff
- const hasData = newItems && newItems.length > 1 && newItems[0].json && Object.keys(newItems[0].json).length > 1;
+ const hasData = newItems && newItems.length > 0 && newItems[0].json && Object.keys(newItems[0].json).length > 1;
```

Left the now-still-dead `newItems.length === 0` check in place — harmless, unreachable before and
after, not worth touching in a one-line fix.

## Verification

Extracted the real `jsCode` from the `Generate Email` node (not a rewritten copy) and ran it in a
live Node.js harness against synthetic item batches, before and after the patch:

| Batch | Before fix | After fix |
|---|---|---|
| 0 items | `has_items: false` (correct) | `has_items: false` (unchanged — correct) |
| **1 item** | `has_items: false` **(bug — email silently skipped)** | `has_items: true`, real HTML email built **(fixed)** |
| 2 items | `has_items: true` (already worked) | `has_items: true` (unchanged — no regression) |

Re-ran the same harness directly against the real, patched `workflow.dev.json` on disk (not a copy)
to confirm the file change itself behaves correctly: 1 item → `has_items: true, total_new: 1`; 0
items → `has_items: false, total_new: 0`; 2 items → `has_items: true, total_new: 2`.

Ran `node scripts/conformance.mjs scripts/regulatory-intel/workflow.dev.json` — valid JSON, passes.

## What's NOT touched

Only the `hasData` boolean guard changed. Scoring (`identifySource()`, urgency/impact
determination), the Layer 2 Ollama re-scoring and fallback-classification steps (C1/C2, B2
addendum), the B5 all-clear email, and every other threshold are untouched. This bug is specific to
the real alert-email path (`Generate Email` / `Send Email Alert`), not the "all clear" status email
(`Send email`), which is a separate node with its own, already-fixed logic.
