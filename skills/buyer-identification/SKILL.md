---
name: buyer-identification
user-invocable: false
description: "Resolve which buyer (institution — school district, college, university, city, county, state agency, hospital, or other organization) a request refers to, and get its Starbridge buyer id. Required FIRST step before any other Starbridge buyer lookup."
when_to_use: "Run whenever a request names or implies a specific institution and you do not already have its buyer id, before buyer-attributes, buyer-summary, contact-search, or document-research. Example triggers — 'how much has Lincoln Public Schools spent', 'tell me about UCLA', 'who is the CIO at Mesa County', 'find RFPs from the city of Austin', or a follow-up using 'they', 'them', or 'that district'. Skip only when the buyer id is already known from earlier in the conversation."
---

# Buyer Identification

Identifies which buyer (institution) the user is asking about. Use it as the first step whenever the user's question involves a specific institution, school, district, agency, or organization.

## When to Use
- The user mentions an institution by name (full, partial, or colloquial)
- The user references an institution indirectly ("how much have they purchased", "what about them")
- Any query that requires knowing which buyer to look up data for

## Tool: `searchBuyers`
Searches the buyer database and returns the top candidates (up to ~10) for you to disambiguate a single named buyer. Use a single search with the best inferred buyer name — the tool handles fuzzy matching for partial or misspelled names. If the user mentions a state or province, set `buyerStateCode` to the matching `StateMapping` enum value (e.g. `California`, **not** `CA` or `US-CA`); the MCP schema lists the valid values and the server resolves them to the ISO 3166-2 code. An unrecognized value is silently ignored and the state filter is dropped, so use the exact enum name.

Matching is by buyer name only (plus the optional `buyerStateCode` filter) — it cannot filter by segment, enrollment, budget, or other attributes, and it is not a list / count / TAM endpoint. For broad "list / count / enrich every buyer matching X" asks, defer to `bulk-request-scoping` rather than improvising a filtered query here.

## Workflow
1. Extract the buyer/institution name from the user's message
2. Call `searchBuyers` with `buyerName` (and `buyerStateCode` when stated)
3. Evaluate the returned candidates. Use the top result without asking only when its `name` matches what the user said (case-insensitive), or the user's phrasing is unambiguously a short form of it (e.g. "MIT" for "Massachusetts Institute of Technology"). Otherwise ask the user to pick, even when there is only one result: colloquial suffixes like "Schools" vs "School District" have produced wrong-buyer answers.
4. If the search returns zero results, retry once with a shorter form of the name (drop "Schools", "District", the state, etc.) before asking.
5. Retain the buyer's `id`, `name`, `url`, and `stateCode` for subsequent tool calls
6. If no candidate is a confident match, do NOT pass a guessed or empty buyer id to downstream tools. Tell the user plainly that no Starbridge buyer matched, echo the name (and state, if any) you searched, and ask which institution they mean — the full name and, if known, which state. When reconciling an external list (e.g. CRM rows), re-run `searchBuyers` with the row's state and corrected name; if there is still no confident match, flag the row as no-match rather than mapping it to a low-confidence buyer.

Once the buyer id is resolved, continue with the skill that matches the question — `buyer-signals` for recent activity, `buyer-summary` for strategy, `buyer-attributes` for a single metric, `document-research` for evidence, and `contact-search` for people.

If the request implies resolving many buyers at once (a pasted list, "all districts in X", a bridge or CRM export), defer to the `bulk-request-scoping` guardrail to bound scope before running per-buyer lookups.

## Example Clarification Response
> "I found <N> institutions that might match. Could you clarify which one you mean? Listing them: …"
