# B5 — All-clear email under-reports monitored sources: verification & fix

**Date:** 2026-09-29 · **Workflow:** `workflow.dev.json` (`Send email` node)

## The bug

The "all clear / no new items" status email (sent when a run finds nothing high-priority) has a
static "Monitored Sources" section listing what the pipeline watches. It listed only **4 of the
5** real feeds:

- 📰 SEC Press Releases
- 📋 Federal Register
- ⚖️ FINRA Enforcement
- 📊 CFTC Regulations

**Investment Advisor Rules** — one of the workflow's 5 real `rssFeedRead` source nodes — was
missing from both this grid and the summary prose line ("...latest scan of SEC, FINRA, CFTC, and
Federal Register feeds."). This is a static, hand-written card that was never updated when the
5th feed was added to the pipeline; it silently under-reports what the system actually monitors on
every "all clear" day.

## Why this matters

This email is the fellow's only signal on a quiet day. If it undercounts the monitored sources, a
reader has no way to know from the email alone that Investment Advisor Rules items were checked at
all that run — it looks like a 4-source system, not a 5-source one.

## The fix

Added the missing card to the grid and the missing feed name to the prose line, matching the exact
node name from the real workflow (`Investment Advisor Rules`):

```html
<div style="padding: 15px; background: #f9fafb; border-radius: 6px; border: 1px solid #e5e7eb;">
  <div style="font-weight: 600; color: #374151; margin-bottom: 4px;">💼 Investment Advisor Rules</div>
  <div style="font-size: 13px; color: #6b7280;">Registered Investment Advisers</div>
</div>
```

## Verification

- Extracted the real `html` parameter from the `Send email` node post-fix and regex-matched every
  source-card label against the workflow's actual `rssFeedRead` node names.
  - Grid now shows: SEC Press Releases, Federal Register, FINRA Enforcement, CFTC Regulations,
    Investment Advisor Rules — **5 of 5**, exact match to the 5 real RSS feed nodes in
    `workflow.dev.json` (`Federal Register - Securities`, `SEC Press Releases`,
    `FINRA Enforcement News`, `CFTC Regulations`, `Investment Advisor Rules`).
  - Prose line now reads "...latest scan of SEC, FINRA, CFTC, Federal Register, and Investment
    Advisor feeds." — all 5 named.
- Ran `node scripts/conformance.mjs scripts/regulatory-intel/workflow.dev.json` — valid JSON.

## What's NOT touched

This is a display-only fix to a static status email. No scoring, classification, or alert-routing
logic changed — `identifySource()`, `determineImpactLevel()`, and every threshold from A7/B2/B3/B4
are untouched. Confirmed while investigating this: the earlier `FINDINGS.md` note "apply A4/B4 to
Generate Email" was stale — both the HTML-escape (`esc()`) and the aligned `>6` threshold were
already present in `Generate Email` and `Generate HTML Report` from the very first hardening
commit (`fa88e05`), before `FINDINGS.md` was even written. No action needed there; documented here
so it isn't re-investigated as if still open.
