---
name: document-research
user-invocable: false
description: "Search a buyer's internal documents — RFPs, board-meeting minutes, strategic plans, procurement records, contracts, and line items — plus buyer-scoped web sources, for detailed, evidence-backed answers."
when_to_use: "Use for history, vendors, technology, contracts, spend detail, and anything needing document evidence rather than a standardized score. Example triggers — 'what have they purchased', 'what vendors do they use', 'find their RFPs', 'what did the board discuss about X', 'do they have a contract with Y', 'what is in their strategic plan'. Use buyer-attributes instead for a single pre-computed metric, and buyer-summary for a high-level narrative. Run buyer-identification first if the buyer id is unknown."
---

# Document Research

Searches internal documents and public web sources for information about a buyer. Provides access to RFPs, board meeting minutes, strategic plans, procurement records, contracts, and buyer-scoped web research.

## When to Use
- Questions about what a buyer has discussed, planned, or decided
- Historical information, meeting minutes, or strategic plans
- Procurement history, contracts, or purchase details
- Research questions requiring document analysis
- Questions that can't be answered with structured attributes alone

## Tools

### `researchBuyerFiles`
Searches internal documents (RFPs, board meetings, procurement data). Returns matching files with summaries and highlights. Procurement records are typically the most reliable source for vendors, technology, and contracts. This — not the web — is the authoritative first stop for buyer research.

### `viewFileContents`
Returns token-bounded *excerpts* of one or more files — each file is truncated to roughly its first N tokens of parsed markdown, and later files are dropped if the total response budget is hit. Accepts `fileId` and/or `opportunityId` (pass an `opportunityId` to pull every file on that opportunity). Use when `researchBuyerFiles` highlights aren't enough for an exact quote or number. It is NOT the complete file, and it is the most of a file you can read — if the user needs the full document, give them the link from `getFileDownloadLinks`.

### `getFileDownloadLinks`
Returns a short-lived (≈1-hour) signed URL to the original file plus a parsed-markdown link, for when the user wants the entire document or the source file itself. The `fileId` is the per-file `id` from `researchBuyerFiles`. Hand the URL to the user; do not fetch either link and read its contents into your context — both files can be very large and neither is size-bounded. The response also includes `contentType` and `originalSizeBytes` (may be null for older files) so you can tell the user what they're opening, and `markdownLink` can be null when the file isn't yet processed.

### `getOpportunityLineItems`
Retrieves line items from contracts or purchase orders. Pass the `opportunityId` (not the file id). Line items are also where **reseller and cooperative** information lives; name them explicitly when the user asks about procurement paths or how to sell into this buyer.

### `runBuyerWebResearch`
Internal files (`researchBuyerFiles` + `viewFileContents`) are the authoritative, more reliable source — exhaust them first. Only call `runBuyerWebResearch` when the answer genuinely requires external context that wouldn't appear in RFPs, board minutes, contracts, or strategic plans: recent news, press releases, or third-party coverage. Returns buyer-scoped web results (URL, title, excerpts) as source candidates. For recency/news questions, the `buyer-signals` feed is the preferred first stop.

## Workflow
1. For factual questions (vendors, technology, contracts), start with `researchBuyerFiles` — procurement records are the most reliable source. When the question needs several independent searches (different document types, acronym variants, separate topics), issue them together in one turn instead of one per turn.
2. Only if internal files don't answer the question — and it genuinely needs external/press coverage — call `runBuyerWebResearch`; do not run it in parallel with the first internal search. (For recency/news, prefer the `buyer-signals` feed.)
3. Review summaries and highlights from search results. Identify whose systems, policies, or actions each passage describes: a vendor proposal, counsel privacy appendix, or service-provider security plan attached to a buyer's meeting packet is evidence about that third party unless the text independently establishes buyer adoption. Keep that distinction in your answer; do not infer the buyer's stack, incident, or purchase intent from the attachment alone.
4. Use `viewFileContents` for longer excerpts, `getFileDownloadLinks` to give the user the full source file, or `getOpportunityLineItems` for purchase-order line items, when highlights aren't enough. Open every file or opportunity you have already chosen in the same turn.
5. Synthesize findings into a comprehensive answer

## Starting from an opportunityId
If you already have an `opportunityId` (e.g. from a Bridge row in the `bridges` skill, or a buyer signal) you do NOT need `researchBuyerFiles` first. Call `viewFileContents` with that `opportunityId` to pull token-bounded excerpts of all files on the opportunity (use `getFileDownloadLinks` if the user wants the full unbounded file), or `getOpportunityLineItems` with the `opportunityId` for its purchase-order line items.

## If Initial Search Yields No Results
Do NOT give up after one search attempt. Try:
1. Different keywords (synonyms, abbreviations, full names)
2. Broader search terms, then narrow down
3. Different document types (board meetings often discuss technology decisions)
4. A wider date window, or none, if you applied a date filter

Turn to `runBuyerWebResearch` only under the rule above: when the question genuinely needs external context such as news or press coverage, not as a generic fallback for an empty internal search.

Only respond with "no information found" after exhausting multiple search strategies.

## Document Type Selection

| Query Type | Primary Sources | Examples |
|------------|-----------------|----------|
| Strategic / Discussion | Board meetings, strategic plans | "plans for", "discussed", "strategy" |
| Procurement / Usage | Procurement data | "purchased", "licenses", "spending" |
| Recent / Time-based | All types | "recent", "latest", "this year" |

## Recency and date filters

"Recent" means something different for each document type, and the date-filter inputs on `researchBuyerFiles` (`effectiveDateGte` / `effectiveDateLte`) and sort fields (`OpportunityFromDate`, `OpportunityUntilDate`) map to *different underlying dates* depending on the type:

- **Meetings / strategic plans:** "from date" = when the document was posted; "until date" is not meaningful.
- **RFPs:** "from date" = posted date; "until date" = due date.
- **Procurement (POs and contracts):** "from date" = effective date; "until date" = expiration / end date.

Because one filter parameter spans all three, the same numeric threshold means different things across types. Do not apply one date filter across mixed types in a single call.

When the user's question implies recency, use these starting defaults — adjust if the initial search returns nothing or the user asked for historical context:

| User asks about | `opportunityTypes` | Default filter / sort |
|---|---|---|
| Meetings, strategic plans, "what did they discuss / decide / plan" | `BoardMeeting_StrategicPlan` | `effectiveDateGte` ≈ 1 year ago; sort `OpportunityFromDate DESC` |
| RFPs — active, open, or recent | `RequestForProposals` | sort `OpportunityUntilDate DESC`; no date filter. Results include both future-due and recently-expired RFPs — distinguish them in your answer by comparing each `OpportunityUntilDate` to today's date |
| Current / active vendor contracts | `ProcurementData` | sort `OpportunityUntilDate DESC` to surface contracts ending furthest in the future. **Do not** filter by `effectiveDateGte` — multi-year contracts signed years ago may still be active |
| Recent purchases, spending, procurement activity | `ProcurementData` | `effectiveDateGte` ≈ 2 years ago; sort `OpportunityFromDate DESC` |

Defaults are a starting point, not a hard rule. Widen or drop the filter if the search returns too little, the user explicitly asks for historical context, or you have reason to believe the relevant document is older (e.g. a 5-year strategic plan, a long-running contract).

### Always disclose the time window you searched

Whenever a date filter was applied — including one you chose yourself without being asked — tell the user the time window the search covered. One line at the end of your answer is enough, e.g.:

- "(searched meetings posted in the last 12 months)"
- "(searched purchases with effective dates in the last 2 years — older long-running contracts may not appear)"

It's fine to default to a recent window when the user asks for "recent" content; it's not fine to filter silently and present the result as exhaustive. Surfacing the window lets the user re-prompt if the answer they want falls outside it.

## When to View Full File Contents
- Comparisons or rankings across documents (largest, most recent, etc.)
- Specific quotes, numbers, or detailed specifications
- When summaries don't provide enough certainty to answer

Remember `viewFileContents` returns truncated excerpts, and they are all you can read. If the answer depends on a part of the document past the excerpt, say so, and give the user the `getFileDownloadLinks` URL so they can open the complete file.

## Quote and snippet requests
When the user asks for quotes, snippets, source excerpts, or evidence for a specific signal or meeting:
1. Stay anchored to that signal, meeting, file, or document set. Open its source first; do not broaden into unrelated files unless the user asks for broader analysis.
2. Use `viewFileContents` for exact quotes whenever the search result only gives summaries, highlights, or relevance explanations.
3. Quote only text that appears in the underlying document or transcript. Never present search-result summaries, relevance explanations, or your own analysis as direct quotes.
4. If you cannot reach the underlying text, say so and paraphrase with the source name instead of adding quotation marks. That is a limitation to state, not a reason to launch a broader search.
5. Lead with the requested quotes; keep analysis secondary.

## Procurement-path questions
When the user asks how a buyer procures, what they've bought, or how to sell into them:
1. Pull the procurement-difficulty score first (`buyer-attributes`; users call it the "procurement hell" score).
2. Search `ProcurementData` with `researchBuyerFiles`, then call `getOpportunityLineItems` for the relevant opportunities.
3. Name the resellers and cooperatives from the line items; they are how sellers actually get in.
4. Use `runBuyerWebResearch` only as a supplemental check after the attribute and procurement files.

## Citing files
- For board-meeting / strategic-plan sources, cite `buyerFiles[].sourceUrl` from `researchBuyerFiles` when present — that is the real citation URL.
- For other document types (RFPs, contracts, purchase orders) there is no citation URL; refer to the document by a human-readable title (and date/type) instead.
- Never surface raw file names, UUIDs, or URL-encoded scraped paths (e.g. the `fileName` / `originalName` fields, or a value like `https_www_youtube_com_watch_v_xyz.vtt`) as links or titles — they are not valid URLs and make it look like the document isn't in Starbridge.

## Tips
- Search both abbreviated and full forms of acronyms (e.g., "FERPA" and "Family Educational Rights and Privacy Act")
- Try refined keywords if initial search doesn't yield relevant results
