# B2 — Source classification fix: verification & measured evidence

**Date:** 2026-08-30 · **Workflow:** `workflow.dev.json` (`Normalize Data` node, `identifySource()`)

## The bug

`identifySource()` labeled *every* `federalregister.gov` item as `'Federal Register - Securities'`
unless a narrow CFTC heuristic matched (link contains `commodity-futures`, or title contains
`cftc`). Two problems, both confirmed live:

1. **The CFTC heuristic never actually matches real CFTC items.** Federal Register document
   permalinks don't embed the agency slug (e.g.
   `federalregister.gov/documents/2026/08/26/2026-17416/swap-execution-facility-...`), and CFTC
   rule titles rarely say "CFTC" literally (e.g. "Swap Execution Facility Order Book Requirement
   for Permitted Transactions"). Live-tested: **12/12** items from the dedicated CFTC Regulations
   RSS feed were mislabeled `'Federal Register - Securities'`.
2. **The "securities+investment" term-search feed (labeled `Federal Register - Securities`) pulls
   in genuinely unrelated agencies** — the search is full-text, not agency-filtered. Live-tested:
   of 146 sampled items, 83 actually came from agencies like the Federal Communications Commission,
   Equal Employment Opportunity Commission, and the DOT Maritime Administration — all mislabeled
   `'Federal Register - Securities'`.

## The fix — read the real issuing agency instead of guessing

Every Federal Register RSS item carries a reliable `<dc:creator>` field naming the actual issuing
agency verbatim (e.g. `Commodity Futures Trading Commission`, `Securities and Exchange
Commission`) — already piped into the code as `creator`, but previously only checked via an
acronym substring (`includes('cftc')`, `includes('sec.gov')`) that never matches Federal
Register's spelled-out agency names, and only in a fallback block the `federalregister.gov`
branch's early `return` made unreachable anyway.

```js
if (lowerLink.includes('federalregister.gov')) {
  const lowerCreator = (creator || '').toLowerCase();
  if (lowerCreator.includes('commodity futures trading commission') || lowerLink.includes('commodity-futures') || lowerTitle.includes('cftc')) {
    return 'CFTC Regulations';
  }
  if (lowerCreator.includes('securities and exchange commission')) {
    return 'Federal Register - Securities';
  }
  if (lowerCreator.includes('financial industry regulatory authority')) {
    return 'FINRA Enforcement News';
  }
  // dc:creator reliably carries the issuing agency's full name (verified live 2026-08-30);
  // anything not SEC/CFTC/FINRA is a real non-financial agency, not "Securities"
  if (creator && creator !== 'Unknown') {
    return `Federal Register - ${creator}`;
  }
  return 'Unknown Source';
}
```

## Live verification (2026-08-30)

Extracted `identifySource()` (old and new versions) and ran both against live RSS pulls from all
5 feeds (script: ad hoc, not committed — see method below):

| Feed | Items sampled | Reclassified | Notes |
|---|---|---|---|
| Federal Register - Securities (term search) | 146 | **83** | Now labeled by real agency (FCC, EEOC, DOT-Maritime, etc.) instead of a blanket "Securities" |
| CFTC Regulations (dedicated feed) | 12 | **12** | 100% were mislabeled "Federal Register - Securities"; 100% now correctly "CFTC Regulations" |
| SEC Press Releases (native feed) | 25 | 0 | No change — already correctly classified via `sec.gov` link domain, unaffected by this fix |
| FINRA (Google News) | 100 | 0 | No change — classified via title heuristic, untouched by this fix |
| Investment Advisor Rules (Google News) | 100 | 0 | No change — same as above |

**Zero regressions** on the three feeds that were already classifying correctly.

## Scope — what this does NOT fix

- **The 21 "Unknown Source" items (Google News fallthrough)** are not addressed here: Google News
  RSS items carry no `dc:creator` (only a publisher `<source>` tag, e.g. "Mayer Brown" —
  irrelevant to agency classification) and their headlines sometimes don't literally contain
  "FINRA"/"SEC"/"investment adviser"/"CFTC"/"securities"/"commodity", so no reliable signal exists
  to close this gap without a fuzzier (and less verifiable) matching approach. Left open. See the
  2026-10-06 addendum below for a partial follow-up.
- **Noise from the loose "securities+investment" term search** (C1, frozen as the Layer-2
  benchmark baseline) is unchanged — this fix corrects the *label* on non-financial items, it does
  not filter them out.

## Verification method

Extracted the `identifySource()` function (old and new) into a standalone Node script; fetched all
5 live RSS feeds; ran both versions against every item and diffed the labels. No live DB write —
classification-only comparison.

---

# Addendum (2026-10-06) — LLM fallback classification for the residual "Unknown Source" items

**Date:** 2026-10-06 · **Workflow:** `workflow.dev.json` · **DB:** local `mycroft_intelligence` @ `localhost:5431`

## Background

Two fixes landed since the B2 fix above narrowed the "Unknown Source" gap without closing it:

- The feed-agnostic synonym fix ("The Synonyms the Classifier Never Learned", 2026-09-07) added
  `adviser`/`RIA`/`broker-dealer`/`Reg BI` title-matching for Google News items, explicitly
  reporting 10 of 18 recovered and leaving 8 open as a deliberate tradeoff — more keyword rules
  would risk false-positiving on unrelated content.

## What this verifies

Re-ran the live `identifySource()` function (extracted directly from the current
`workflow.dev.json`, not a copy) against the actual 21 `Unknown Source` rows still in the DB:

**13 of 21 are already resolved by the current keyword rules** (the two fixes above cover more
than the original `FINDINGS.md` snapshot). The genuinely unresolved residual is **8 items**,
matching the synonym fix's own honest accounting:

```
Understanding the Regulations of 24-Hour Trading - Investopedia
From Inquiry to Response: What to Do When Regulators Come Knocking for Text Messages - corporatecomplianceinsights.com
Ropes & Gray's Investment Management Update January – February 2026 - Ropes & Gray LLP
What Are Regulatory Assets Under Management (RAUM)? - SmartAsset
Capital Markets & Governance Insights - July 2025 - Ropes & Gray LLP
Know Your Client (KYC): Key Requirements and Compliance for Financial Services - Investopedia
Recent Enforcement Actions Define the "Person" that Participates in a Partial Tender - JD Supra
When Two Become One: Navigating the Complexities of Operational Integration - Lowenstein Sandler LLP
```

These are generic explainer/digest headlines — no fixed keyword list resolves them without
false-positiving on unrelated content, which is exactly the brittleness the 2026-09-07 STEM video
("Embeddings: How AI Tells Similar From Different") argued keyword matching can't escape.

## The fix

Added a new Code node, **`Layer 2 - LLM Source Classification`**, inserted between `Normalize
Data` and `Filter Valid Content`. Design, matching the existing `Layer 2 - LLM Re-Score` node's
conventions:

- Only calls the LLM (local Ollama, `llama3.2:3b`) for items where `source_feed === 'Unknown
  Source'` — every other item passes through completely untouched.
- Prompts the model to pick one of the 5 known categories (or `Unknown Source`) based solely on
  the headline.
- Validates the model's reply against the fixed category list; an invalid/unexpected reply is
  treated the same as `Unknown Source` (no silent mis-classification).
- **Fail-open**: any network/parse error leaves the item's `source_feed` exactly as it was
  (`Unknown Source`), logs `llm_source_error`, and never blocks the run.

## Live verification (not simulated)

Built a Node.js harness that runs the *exact* extracted node code (not a rewritten copy) against
the real 21 DB rows, through the full two-stage pipeline (keyword rules first, then the new node):

```
After current keyword rules: 13 already resolved, 8 still Unknown Source
Items with already-known source_feed incorrectly mutated: 0 (must be 0)
Layer 2 LLM source classification: 8 unresolved items reviewed, 1 resolved, 0 errors
  RESOLVED [544] "Recent Enforcement Actions Define the "Person" that Participates in a Partial
  Tender - JD Supra" -> FINRA Enforcement News
```

**Honest result: 1 of 8 resolved.** The model correctly declined to guess on the remaining 7
rather than force a label onto genuinely ambiguous/generic content (an Investopedia KYC
explainer, law-firm monthly digests, a 24-hour-trading explainer) — this is the intended
behavior, not a shortfall in the prompt. Some of this backlog isn't a classification bug; it's
content that doesn't actually belong to any of the 4 tracked categories.

**Fail-open path separately verified**: pointed the extracted node at an unreachable Ollama URL
and re-ran it. The already-classified item passed through byte-identical; the `Unknown Source`
item stayed `Unknown Source` with `llm_source_error: "fetch failed"` logged — the run does not
break.

## Honest limitations

- Net yield is modest (1 new item resolved per this sample of 8) — this is a narrowing of an
  already-small residual, not a full close of B2.
- Single-pass, no retry; a transient Ollama failure on a classifiable item simply leaves it
  `Unknown Source` for that run (same fail-open tradeoff as `Layer 2 - LLM Re-Score`).
- Small sample (8 items) — yield on a larger, more varied set of future `Unknown Source` items is
  unverified.
- Adds one Ollama call per `Unknown Source` item per run (bounded — only ever a handful per run
  based on observed volume, same per-run latency tradeoff already accepted for `Layer 2 - LLM
  Re-Score`).

## Conformance

`node scripts/conformance.mjs scripts/regulatory-intel/workflow.dev.json` — clean after the
change.
