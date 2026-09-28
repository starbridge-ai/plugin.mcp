---
name: buyer-summary
user-invocable: false
description: "Get a synthesized narrative of a buyer's tracked activity — current priorities, momentum, risks, opportunities, and outreach-ready talking points grounded in Starbridge data."
when_to_use: "Use as the default grounding layer for strategy, call prep, and 'what do we know about them' questions. Example triggers — 'what are their priorities', 'give me talking points for X', 'help me prep for a call with this district', 'what is going on with them', 'how should I approach them'. Prefer this before document-research for strategy and outreach framing. For dated 'what's new / recent / any new signals' questions, prefer buyer-signals (the raw dated feed); use this skill for the synthesized strategic narrative. Use buyer-attributes instead for a single standardized metric, and document-research when the user needs underlying evidence, exact procurement details, or verification. Run buyer-identification first if the buyer id is unknown."
---

# Buyer Summary

Retrieves a buyer summary synthesized from tracked buyer activity and available signals. It is the fastest way to understand the buyer's current strategy, momentum, risks, open questions, and where to focus outreach.

## When to Use
- The user wants to know the buyer's strategy or top priorities
- The user asks for talking points, email angles, call preparation, or other outreach
- The user asks what information we have about a buyer across the data we've collected
- The user explicitly asks for the buyer summary

## Tool: `getBuyerSummary`
Returns a structured summary grounded in tracked account activity and available signals. The `summary` field has three sections — `whatsRecent`, `theBiggerPicture`, and `areasForDeeperDiscovery` — and each section's `sources` carry `triggerId`/`entryId` back-pointers you can use to drill into a citation. Requires the `buyerId` path parameter.

It generates on demand when the cached summary is missing or stale, so the first call for a buyer can be noticeably slower — set expectations with the user rather than assuming the tool stalled. If `summary` is `null` with `needsRegeneration: true`, a (re)generation is in flight: treat it as "not ready yet," not "no data," and retry shortly. A 404 (`resource.buyer.summary.empty`) means no Bridge in the org is configured to track this buyer, so there is nothing to summarize — tell the user and suggest setting up a Bridge that covers the buyer (they may still have documents and a web footprint, so you can still offer `document-research` or `buyer-attributes`). For other transient failures, fall back to `document-research`.

## Workflow
1. Ensure the buyer has been identified first via `buyer-identification`
2. Call `getBuyerSummary` with the buyer id
3. Use the summary as the default grounding layer for strategy, buyer priorities, and outreach prep
4. If the user needs underlying evidence, exact procurement details, or deeper verification — or the summary is unavailable (handle per the tool notes above) — activate `document-research`
5. If the user wants drafted outreach, use the summary to shape the message before invoking an outbound email skill
6. To verify or expand a specific point, take the `triggerId` from that section's `sources` (it is the bridge id), pass it as `bridgeId` to `listBridgeRows` and find the row whose `rowId` equals the source's `entryId` (see `bridges`). There is no `rowId` filter, so narrow with a `buyerId` `Equals` filter term and page until the row appears; for the underlying source file, hand off to `document-research`

## Important
- Treat the summary as a synthesis of tracked activity and signals, not as permission to invent details beyond it
- Prefer this skill before `document-research` for strategy or outreach questions
- For dated "what's new / recent" questions, prefer `buyer-signals`; use this skill for the synthesized narrative
- On failure, degrade gracefully: a 404 (`resource.buyer.summary.empty`) means no Bridge tracks this buyer — suggest setting one up rather than silently retrying; for other transient failures, continue with `document-research`
- Use the summary's sections directly: `whatsRecent`, `theBiggerPicture`, and `areasForDeeperDiscovery`
