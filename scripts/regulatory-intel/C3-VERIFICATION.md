# C3 — Layer 2 LLM's corrected severity silently discarded by Generate Email: verification & fix

**Date:** 2026-10-27 · **Workflow:** `workflow.dev.json` (`Generate Email` node)

## The bug

`Layer 2 - LLM Re-Score` (added in C2) reviews any item with `urgency_score > 6` and writes back
`llm_reviewed`, `llm_verdict`, `llm_corrected_impact_level`, and `llm_downgrade` — the whole point
being that the LLM can correct a keyword-scorer severity that's wrong (e.g. downgrade a
"Critical"-flagged item, or recategorize it to "High" without rejecting it outright).

But `Generate Email` — the node that actually buckets items into the Critical/High sections of the
alert email and counts them — never reads `llm_corrected_impact_level` anywhere:

```js
const criticalItems = newItems.filter(i => i.json.impact_level === 'Critical');
const highItems = newItems.filter(i => i.json.impact_level === 'High');
```

Only the original keyword-based `impact_level` is used. `High Priority Filter` only consumes the
binary `llm_downgrade` flag (per C2), so an item the LLM *confirms* (doesn't downgrade) but
*recategorizes* (e.g. Critical -> High) still passes the filter — and then lands back in the email
under its stale, pre-LLM label. The correction is computed, written to the item's JSON, and then
never looked at again.

## Why this matters

This is the third fix in this same email-generation node (after B5's under-reported sources and
B6's dropped-on-exactly-1-item bug), and the second bug specifically in the Layer 2 LLM pipeline
C2 introduced. It means the LLM review step's own stated purpose — correcting severity, not just
confirming/rejecting — has never actually worked for any item that reaches this node, since the
day C2 shipped. An operator reading the email sees exactly the same bucketing they would have seen
with no LLM review at all, as long as the LLM doesn't downgrade the item out entirely.

## The fix

`workflow.dev.json`, `Generate Email` node — compute an effective severity that prefers the LLM's
correction when one exists, falling back to the keyword scorer's original label:

```diff
 const highPriority = newItems.filter(i => i.json.urgency_score > 6);
-const criticalItems = newItems.filter(i => i.json.impact_level === 'Critical');
-const highItems = newItems.filter(i => i.json.impact_level === 'High');
+
+const effectiveImpact = (item) => (item.json.llm_reviewed && item.json.llm_corrected_impact_level)
+  ? item.json.llm_corrected_impact_level
+  : item.json.impact_level;
+
+const criticalItems = newItems.filter(i => effectiveImpact(i) === 'Critical');
+const highItems = newItems.filter(i => effectiveImpact(i) === 'High');
```

Items never sent to the LLM (`llm_reviewed: false`, below the `urgency_score > 6` review
threshold) are unaffected — they fall through to the original `impact_level`, exactly as before.

## Verification

Extracted the real, patched `jsCode` from the `Generate Email` node (not a rewritten copy) and ran
it in a live Node.js harness (`$("High Priority Filter").all()` shimmed) against a 3-item batch
representing every real case that reaches this node:

| Item | `impact_level` (original) | LLM review | Before fix | After fix |
|---|---|---|---|---|
| 1 — confirmed, recategorized | Critical | reviewed, confirmed, corrected to High | counted **Critical** (bug) | counted **High** (fixed) |
| 2 — confirmed, no change | High | reviewed, confirmed, no correction | counted High | counted High (unchanged) |
| 3 — never reviewed | Critical | not reviewed (`urgency_score <= 6`) | counted Critical | counted Critical (unchanged) |

Before fix: `critical_count: 2, high_count: 1` (item 1 wrongly counted Critical). After fix:
`critical_count: 1, high_count: 2` (item 1 correctly counted High; items 2 and 3 unchanged,
confirming no regression for already-correct cases). Re-ran against the real patched
`workflow.dev.json` on disk, not a synthetic copy — same result.

Ran `node scripts/conformance.mjs scripts/regulatory-intel/workflow.dev.json` — valid JSON, passes.

## What's NOT touched

Only the `criticalItems`/`highItems` bucketing in `Generate Email` changed. `Layer 2 - LLM
Re-Score`'s own logic (C2), `High Priority Filter`'s `llm_downgrade` gate, the B5 all-clear email,
and the B6 `hasData` guard are untouched. Items that were never LLM-reviewed behave identically to
before. `critical_count`/`high_count` in the node's own JSON output change to match the corrected
buckets — any downstream consumer of those two fields (if one exists outside this node) would see
the corrected counts, which is the intended effect, not a side effect.
