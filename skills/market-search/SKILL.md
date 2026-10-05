---
name: market-search
user-invocable: false
description: "Search procurement records across many buyers at once — a territory, a state, a buyer type, or every buyer Starbridge tracks — for topics discussed, open RFPs, grants and budget funding, vendor footprint and spend, and the cooperative contracts and resellers buyers purchase through."
when_to_use: "Use for questions with no single institution. Example triggers — 'which districts in Texas discussed AI tutoring', 'open RFPs we could bid on', 'who buys from <vendor> and for how much', 'spend on <category> in the Southeast', 'which coops do cities use for <product>'. For one named buyer use document-research; for account scores and configured competitor presence use account-scoring."
---

# Market Search

Answers questions about many buyers from Starbridge's procurement corpus: RFPs, purchase orders, contracts, board meetings, budgets, and plans. Internal records are the source of truth here; the public web only explains context the records cannot (what a cooperative is, a vendor's product line).

## Scope the buyers first

Every question here has a buyer population. Resolve it before searching and state it in the answer.

- **"My territory", "my accounts", "my market"** → `getCurrentUser`'s `defaultBuyerListId`, passed as `buyerListId`. When it is null, say no territory is set and ask which list or region to use; do not substitute the organization's scoring market.
- **A named saved list** → find its id with `listBuyerLists`, pass as `buyerListId`.
- **A region or buyer type** ("Florida cities", "Texas school districts") → `buyerTerms`, with field and operation keys from `getBuyerSchema` (`forSearch=true`). Never send both `buyerListId` and `buyerTerms`.
- **No population named** → search every buyer and say so.

`searchOpportunities` takes `buyerListId` or `buyerTerms` directly. `searchPurchaseOrders` takes only `buyerIds`: resolve the list or criteria first with `searchBuyersByCriteria` (`idsOnly=true`, up to 10,000 ids), or use `searchOpportunities` with `opportunityTypes` `PurchaseOrder` and `Contract` instead. A purchase search sent without the population covers every buyer.

## Pick the tool by question

| Question | Tool | Key arguments |
|---|---|---|
| Who discussed a topic, planned it, or budgeted for it | `searchOpportunities` | `queries`; `opportunityTypes` `BoardMeeting`, `BoardMeetingAgenda`, `BoardMeetingAgendaPacket`, `StrategicPlan`, `CapitalImprovementPlan`; `fromDateRelativePeriod` |
| New grants or budget funding | `searchOpportunities` | `opportunityTypes` `Budget`, `BudgetAmendment`, `ProposedBudget`, `AnnualFinancialReport` plus the meeting types above (awards are usually voted on in meetings) |
| Open RFPs to bid on | `searchOpportunities` | `opportunityTypes` `Selling`; `untilDateRelativePeriod` `Future` |
| What buyers bought in a category, and for how much | `searchOpportunities` with `opportunityTypes` `PurchaseOrder`, `Contract` and the population; `searchPurchaseOrders` for line-level detail | `searchPurchaseOrders`: `name` (line-item substring), `buyerIds`, `sellerIds` via `findSellers`, `sortKey` + `sortDirection` |
| Which buyers in a region use a vendor, and contract amounts | `searchOpportunities` | `queries` set to the vendor name, `opportunityTypes` `PurchaseOrder`, `Contract`, the population |
| Which of our target-market accounts use a configured competitor or partner | `account-scoring` presence bridges | Covers only the organization's target market, not every buyer in a region |
| How a vendor structures contracts over time, across all buyers | `searchVendorPurchaseOrders` | `vendorName`, `alias`, `productName`, `lookbackYears` up to 10, `limit` up to 80 |
| Which cooperatives or resellers buyers purchase through | `searchPurchaseOrders` and `getOpportunityLineItems` | Read seller names and line-item text for the vehicle (Sourcewell, OMNIA, NASPO ValuePoint, TIPS, BuyBoard, E&I, GSA) and the reseller |
| Whether the calling organization can sell through a vehicle | `listContractVehicles` | — |

`getOpportunity` returns one record in full (summary, files, line items); `scoreOpportunities` re-ranks records already found against the user's criteria.

## Searching with `searchOpportunities`

- Send two to four short, distinct `queries` rather than one long sentence. Include synonyms and acronyms ("phishing", "email security", "business email compromise").
- `minimumMatchScore` defaults to 3. Use `freeFormScoringPrompt` to describe what counts, drawn from the organization's business context when the request is "relevant to what we sell".
- Paging streams without a total. An empty page is not the end when `queries` is set — fetch one more page before stopping.
- **Discussed but no RFP yet:** find the discussions, then run a second search with `opportunityTypes` `Selling` scoped to those buyers (`buyerTerms` on `Id`) and drop any buyer with a matching RFP. Say the exclusion covers only RFPs Starbridge has.

## Vendor footprint in a region

Call `searchOpportunities` with `queries` set to the vendor name, `opportunityTypes` `PurchaseOrder` and `Contract`, and the region as `buyerTerms` (or the territory as `buyerListId`). With only purchase types this runs a seller-context search that matches the vendor's many recorded spellings ("GRANICUS, LLC DBA GRANICUS", "12892 - GRANICUS, LLC"); it takes no `sort` or `freeFormScoringPrompt`. Use `pageSize` 50 and page until a page comes back empty, then group by buyer. This covers every buyer in the region, not only the organization's target market. `findSellers` returns each spelling as a separate seller, so use it only when the user names a specific seller record.

`searchVendorPurchaseOrders` is a sample, not a footprint: it has no region filter and returns at most 80 records across all states, newest buyers first. Use it to read a vendor's contract patterns, and call any regional count drawn from it partial.

## Broaden before reporting nothing

An empty or thin first result is a reason to search again, not an answer. Work down this list, up to two more searches:

1. Rephrase `queries` with synonyms, acronyms, and the vendor or product names involved.
2. Add the neighboring document types from the table.
3. Lower `minimumMatchScore` to 2.
4. Widen or drop the date window.
5. Widen the buyer scope one step (list → state, state → all buyers) and say you did.

When the searches stay empty, report what you covered: the buyer scope, document types, date window, and queries. Do not conclude that nothing happened — only that Starbridge's records in that scope do not show it.

## Units and citations

- Purchase-order and opportunity amounts are whole currency units (US dollars): `58166` is $58,166. Do not divide by 100. Identical amounts on the same date for one buyer are usually one purchase order reported twice.
- Name each record by buyer, document type, title, and date next to the claim it supports. Keep record identifiers out of the prose.
- State the population and window the answer covers, and whether it is complete or a sample.
