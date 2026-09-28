---
name: bulk-request-scoping
user-invocable: false
description: "Recognize broad, Clay-style bulk-enrichment requests (enrich every row / contact / account) and scope them to a Starbridge-shaped task — narrow, sample, or hand off — before grinding through many slow, approval-gated tool calls."
when_to_use: "Run as a guardrail BEFORE starting a large enrichment loop. Example triggers — 'enrich all <N> rows / contacts', 'bulk enrich this list', 'find contacts for every buyer in this bridge', 'do this for all my accounts', 'run a summary on each of these', or any request implying many buyers each needing a slow per-buyer call (contact search, buyer summary, or web research). This skill shapes the task and sets stop-conditions; it does not produce the enrichment itself — hand off to contact-search / buyer-summary / document-research once scoped."
---

# Bulk Request Scoping

A guardrail for broad, Clay-style bulk-enrichment asks. Starbridge MCP is built for **targeted, per-buyer research** and for **operating on existing Bridge data** — not for bulk table enrichment across hundreds of rows. Slow tools (`searchBuyerContacts`, `getBuyerSummary`, web research) chained across many buyers cause approval fatigue and timeouts. This skill catches that shape early and steers it.

## When to Use
- The request implies *many buyers × a per-buyer enrichment*, especially using slow tools
- A user asks to enrich, summarize, or find contacts for "all" / "every" / "each" item in a list or bridge
- Before kicking off a loop that would generate many approval-gated calls

This does **not** apply to small requests (a handful of buyers) — proceed normally there.

## Workflow
1. **Estimate scope.** How many buyers × how many per-buyer enrichments, weighted by tool speed (`searchBuyerContacts` and `getBuyerSummary` are slow). If the source is a bridge, `listBridgeRows` (page 1) gives the row count / `totalPages`.
2. **Decide if it's "large."** Treat it as large when many buyers each need a slow per-buyer call — i.e. the plan would produce tens of approval-gated calls or minutes of chained work. (Express the cost in calls/time, not a fixed magic number.) Small jobs: just proceed.
3. **If large, STOP before looping** and present options:
   - **Narrow** — a named subset, or top-N by a stated criterion (e.g. top 10 by score or recency).
   - **Sample** — run a bounded sample now (e.g. the first 5) so the user sees the shape and cost, then decide whether to continue.
   - **Hand off** — true bulk enrichment belongs in the Starbridge app (a bridge with enrichment columns) or a Clay-style tool. Explain why, and what Starbridge MCP does well instead (targeted research, operating on existing bridge data).
4. **Proceed only on explicit choice.** If the user opts to continue, keep the call count down — request every attribute a buyer needs in one `getBuyerAttributesBulk` call rather than one call per attribute — and batch sensibly.

## Tools (used only after scoping)
- `listBridgeRows` — to size the job when the source is a bridge.
- `getBuyerAttributesBulk` — many attributes for **one** buyer per call. It is not a multi-buyer batch: N buyers still cost N calls, so count them when sizing the job.
- The actual enrichment runs through the relevant skill once scoped: `contact-search`, `buyer-summary`, or `document-research`.

## Important
- This is guidance and stop-conditions, not a producer — once the task is scoped, hand off to the appropriate skill.
- Don't false-stop small requests; the guard is for many-buyers × slow-per-buyer work.
- Be explicit about the trade-off (number of calls, approvals, time) so the user can make the call — don't silently grind through a bad plan.
