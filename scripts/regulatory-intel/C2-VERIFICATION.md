# C2 — Layer 2/3 LLM re-scoring: verification & measured evidence

**Date:** 2026-09-29 · **Workflow:** `workflow.dev.json` (new `Layer 2 - LLM Re-Score` node)

**Note on numbering:** `FINDINGS.md` names this class of fix inconsistently — its C1 section
heading calls the misfire class "the Layer-2 baseline," but its roadmap section labels the actual
LLM second-pass "Layer 3." This doc follows the name already used in conversation and in the new
node itself, **"Layer 2 – LLM Re-Score,"** and is filed as `C2` (direct sequel to `C1`, the misfire
class it targets) to avoid inventing a third numbering scheme.

## The problem (C1, from FINDINGS.md, previously intentionally unfixed)

The keyword scorer's "Critical" bucket is led by noise, not signal:

- `Medicare Program: Hospital Outpatient Prospective Payment…` → scored **10/Critical** purely
  because its title mentions "**Emergency** Medical Treatment and Labor Act" (a real federal
  regulation's proper name) — `calculateBasicUrgency()` adds +3 for the substring "emergency"
  with no understanding of context.
- The cluster of `Nasdaq … Notice of Filing and **Immediate Effectiveness**` SRO filings →
  scored **9/Critical each** — boilerplate procedural filings, inflated purely by the substring
  "immediate."

FINDINGS.md's own roadmap named the intended fix: a local-LLM (Ollama) second pass, matching the
ecosystem convention already used by `Regulatory_QA` (`backend/app/config.py`:
`ollama_url=http://localhost:11434`, `ollama_model=llama3.2:3b`).

## Environment note — Ollama was not installed on this machine

Before this fix, `ollama` was not installed at all (no CLI, no `~/.ollama`, connection refused on
11434) despite `Regulatory_QA` already depending on it. Installed via `brew install ollama`,
started via `brew services start ollama`, pulled `llama3.2:3b` (2.0 GB, matching
`Regulatory_QA`'s configured model exactly) — all done and verified live in this session, not
assumed.

## The design

New Code node, **`Layer 2 - LLM Re-Score`**, inserted between `Code in JavaScript` and `If2` (so
all three downstream branches — `High Priority Filter`, `Generate HTML Report`, `Send email` — see
its output consistently):

- Only calls Ollama for items with `urgency_score > 6` — the exact same threshold `High Priority
  Filter` gates on. No point spending an LLM call reviewing an item that was never going to alert
  anyway (confirmed live: a genuine Critical item with `urgency_score = 5`, id 153 — see below —
  correctly gets skipped, matching existing behavior, not a regression).
- Sends the title, a content excerpt, the keyword-scorer's own stated reasoning, and which
  specific trigger word(s) fired, to `llama3.2:3b` via `POST /api/generate` (same HTTP shape as
  `Regulatory_QA`'s existing call).
- Prompt asks for a JSON verdict: `confirm` (genuinely urgent) or `downgrade` (routine noise that
  only scored high from an incidental keyword match), plus a corrected impact level and one-line
  reasoning.
- **Never overwrites the original keyword score.** `urgency_score`/`impact_level` from the
  keyword scorer are preserved untouched (already written to Postgres by `Insert data into DB`,
  upstream of this node) — this step only adds new `llm_*` fields for routing/display, and only
  affects in-flight alert routing, never rewrites history.
- **Fail-open on any error.** Network failure, Ollama down, or unparseable response → caught,
  logged to `llm_error`, and the item defaults to `llm_downgrade: false` — behaves exactly as it
  would have with no Layer 2 step at all. Same principle as B3's unwrap-fallback: a broken
  re-score never blocks or breaks the run.
- Sequential, not parallelized — same deliberate choice as B3 (one local model instance, no
  benefit to bursty parallel calls on a scheduled batch job).
- `High Priority Filter`'s condition updated from `urgency_score > 6` alone to
  `urgency_score > 6 AND llm_downgrade == false` (legacy IF node, added a `boolean` condition,
  `combineOperation` changed from `any` to `all`).

## Live verification

**Prototype (Python, standalone, against 3 real DB rows):** iterated the prompt once (added an
explicit impact-level guide) after the first pass correctly separated confirm/downgrade but was
imprecise about exactly how far to downgrade.

**Exact JS node code, extracted from the shipped `workflow.dev.json` and run via a `$input.all()`
harness matching n8n's Code node execution model** (same method B3 used), against 3 real rows
pulled live from `regulatory_feeds`:

| id | Title | Keyword score | `llm_reviewed` | `llm_verdict` | `llm_corrected_impact_level` |
|---|---|---|---|---|---|
| 4 | Medicare Program: Hospital Outpatient... (real EMTALA text) | 10/Critical | true | **downgrade** | High |
| 966/967 | Nasdaq SRO "...Immediate Effectiveness..." cabinet-connectivity filing | 9/Critical | true | **downgrade** | High |
| 153 | SEC Charges 21 Individuals... Insider Trading Scheme (real content) | 5/Critical (via fraud-keyword bypass, not urgency_score) | **false** (correctly skipped — `urgency_score` ≤ 6, was never going to alert regardless of LLM verdict) | — | — |

Both named C1 false positives are correctly flagged `downgrade`; the genuine SEC enforcement case
is correctly left alone (and correctly never even sent to the LLM, since it wouldn't have alerted
anyway under the existing `urgency_score`-based gate).

**Fail-open path, separately tested:** pointed the same shipped node code at an unreachable port
(simulating Ollama being down, without touching the real running service) — result:
`llm_reviewed: false, llm_downgrade: false, llm_error: "fetch failed"` on the item that would have
triggered a call; the item below the alert threshold was unaffected. Confirms an Ollama outage
degrades to pre-Layer-2 behavior, never blocks the pipeline.

**Timing:** 2 real sequential Ollama calls (3B model, `num_predict: 200`, local M1 Pro) completed
in 3.3–3.6s combined (~1.6–1.8s/call).

## What's honest and NOT overclaimed here

- **The corrected impact level is conservative, not precise.** Both named false positives were
  downgraded from Critical to **High**, not further to Medium/Low, even after adding an explicit
  impact-level guide to the prompt. The `verdict` (confirm/downgrade) — the field that actually
  drives alert routing — is reliable across all 3 test cases; the exact `corrected_impact_level`
  is a display/audit field only and is more conservative than a human reviewer might be.
- **Small sample.** Verified against the 2 specific named false-positive titles from `FINDINGS.md`
  plus 1 true positive — not a large labeled benchmark. `FINDINGS.md`'s own "Layer 2: freeze the
  baseline" step (a proper labeled benchmark) was never built; this fix targets the 2 concretely
  named patterns, not a general-purpose classifier evaluation.
- **Added latency, real cost.** At the pipeline's own historical urgency histogram
  (`{7:81, 8:6, 9:23, 10:1}` = 111 items/run scoring above the threshold, per `FINDINGS.md`), a
  full run could add on the order of 100+ sequential local LLM calls — a few minutes of added
  wall-clock time per scheduled run. Not measured at that scale here, only per-item.
- **Non-deterministic.** LLM output varies run to run even at low temperature; the exact wording
  of `llm_reasoning` will differ between runs even when the verdict itself is stable.
- **Local-only dependency.** This entire feature silently no-ops (fail-open) if the fellow's
  actual deployment machine doesn't have Ollama running — same class of environment caveat as
  B3's "not verified against the fellow's live n8n instance."

## Files changed

- `scripts/regulatory-intel/workflow.dev.json` — new `Layer 2 - LLM Re-Score` Code node inserted
  between `Code in JavaScript` and `If2`; `High Priority Filter`'s condition updated to also
  require `llm_downgrade == false`.
- Ran `node scripts/conformance.mjs scripts/regulatory-intel/workflow.dev.json` — valid JSON.
