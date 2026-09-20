---
name: account-scoring
user-invocable: false
description: "Read the user's Top Buyers bridge (Account Scoring) and the competitor and complementary-vendor presence bridges it links to — account scores, fit and signal bands, rankings of target accounts, and which accounts a competitor or partner is present in, with contract values and dates."
when_to_use: "Use whenever the user asks about account scores, fit or signal bands, best or hottest accounts, target-account rankings, ICP prioritization, which accounts a competitor or partner is present in, which buyers in a state or of a type use a vendor, a vendor's contract values or dates, a vendor's footprint by state, or anything that ranks, filters, or aggregates buyers in their target market — even if they never say 'Top Buyers' or 'bridge'. For other bridges use bridges; for one institution's own data use buyer-summary, document-research, or buyer-attributes."
---

# Account Scoring (Top Buyers)

Three organization-wide tables, all read with the generic bridge tools. The Top Buyers bridge has one row per buyer in the organization's target market with a locked **Account Score** column (0–100) plus locked **Buyer State Name** and **Buyer Type** columns. Two market presence bridges have one row per buyer × configured competitor or complementary vendor with the vendor's contract data. Top Buyers links to the presence bridges through Link columns.

## Tools

- `listBridges` — `filterType=TopBuyer` finds the Top Buyers bridge (at most one per organization). `filterType=ProductCompetitorPresence` / `CompanionPresence` find the market presence bridges.
- `getBridgeColumnMetadata` — one entry per column. The base shape (`name`, `type`, `fieldFormat`, `key`) is nested under `column`; `columnId`, `linkSettings` and `accountScoringMetadata` (fit columns, signal configuration) sit at the top level. Required before any filter or sort.
- `listBridgeRows` — rows with `filters`, `sorts`, `query`, pagination.
- `getBuyerAttributesBulk` — many attributes (name, type, state, population, …) for **one** buyer per call. It is not a batch join; see Step 3 for the batch join.
- `listRecentBuyerSignals` — recent signal rows for one buyer; the evidence behind a Signals band.
- `searchVendorPurchaseOrders` — procurement records for a vendor across all buyers; the fallback when a vendor is not configured in a presence bridge.

## Step 1 — Locate the bridge and columns

1. `listBridges(filterType=TopBuyer)`. No result → the organization has no Account Scoring set up; say so, do not substitute another bridge.
2. `getBridgeColumnMetadata(bridgeId)`. Record the `columnId` of:
   - **Account Score** (`column.type=AccountScoring`) — the cell value is an object. Never sort or filter on this id: object columns sort by their JSON text, so the call succeeds and returns a wrong order.
   - Its child columns — hidden in the UI but present in the API and in every row: **Account Scoring Score** (number), **Fit Band**, **Signal Band**, **Fit**, **Signals**. Sort and filter with these ids. If an older bridge lacks them, read the score from the object and say that server-side ranking is unavailable.
   - **Buyer State Name** — plain text holding the full state name (`"Florida"`, not `FL`). Filter with `Equals` or `Any`.
   - **Buyer Type** — an enum; `column.fieldFormat` lists the accepted values. The main labels are `City`, `County`, `State Agency`, `State`, `Higher Education`, `School District`, `School`, `Police Department`, `Fire Department`, `Library`, `Special District`, `Federal`. Map the user's words to these ("cities" → `City`, "state agencies" → `State Agency`).
   - Link columns named **Competitive Presence** and **Companion Presence** — `linkSettings.sourceBridgeId` is the presence bridge id.
   - Any customer-added columns (AI analysis, CRM lookups, signal MultiLinks). Read their names; they may answer the question directly.

Filter and sort terms take a bare `columnId` UUID as `field` / `column`. Column names and keys return 400.

## Step 2 — Read rows

Filter server-side; do not page the whole bridge and filter in memory unless the question genuinely needs every row.

- Rank: `sorts=[{column:<Account Scoring Score id>, direction:DESC}]`.
- "Hottest" / "right now" / "active": filter **Signal Band** `Any ["Hot","Warm"]`, then sort by Account Scoring Score DESC. Never sort by a band column; enums sort alphabetically.
- Restrict: `filters.terms` on Buyer Type (`Equals`, or `Any` with an array), Buyer State Name, Fit Band / Signal Band (`Any`), or `buyerId` (`Any` with an array of UUIDs).
- Page with `pageSize` ≤ 20 until `pageNumber == totalPages`. Each Top Buyers row costs one link lookup per Link column, so keep pages small.

Row shape: `rowId`, `buyerId`, `name` (buyer name), `columns` keyed by column **name**. Every cell is `{ value, status, processedAt }`. The child cells give flat values — `columns["Account Scoring Score"].value`, `columns["Fit Band"].value`, `columns["Signal Band"].value` — so read bands from them. The Account Score cell's `value` is

```
{ score,
  fit: { band, shortExplanation, longExplanation: { summary, factors: [{ phaseId, phaseName, explanation }] } },
  signals: { band, shortExplanation, sources, briefing: { overview, opportunities, accountContext } } }
```

`factors[].phaseName` names the fit column that drove the factor. `fit.band` ∈ Weak, Moderate, Strong, Ideal. `signals.band` ∈ NoSignals, Blocked, Cool, Warm, Hot; `Blocked` means a competitor already holds the account.

A `null` Account Score is not zero. It means the row has not been scored yet, or possibly that the caller is a Viewer and the organization has not shared scores with Viewers. Say which is likely rather than reporting 0.

## Step 3 — Vendor presence

Link cells on Top Buyers return at most 3 `previewRows` plus a `total`. When the count matters or `total > 3`, read the presence bridge itself:

1. `getBridgeColumnMetadata(presenceBridgeId)` for the ids of **Vendor**, **Vendor Presence** (boolean "Present"), **Vendor Presence Summary**, **Estimated Annual Contract Value**, **Estimated Total Contract Value** (when present), **Estimated Contract End Date**, **Contract Term**, **Inferred**, **Inference Summary**, **Sources**, and on the competitor bridge **Products**.
2. `listBridgeRows(presenceBridgeId)` with `Vendor Presence Equals true`, plus `Vendor Equals <name>` (or `Any [...]`) or `buyerId Any [...]` as needed. Presence rows have no Link columns, so `pageSize` 100 is fine here.
3. Presence rows carry no geography or buyer type; join back to Top Buyers as described below. Use `getBuyerAttributesBulk` only for a handful of buyers; it is one call per buyer.

Amounts are USD in cents. Contract End Date is an estimate; when `Inferred` is true say so and cite `Inference Summary`. `Sources.value.relevantOpportunities[]` lists the underlying purchase orders and contracts with `title`, `opportunityType`, `startDate` and `endDate`; use those dates for "most recent contract" questions. There is no signing date column.

Only vendors configured by the organization appear in presence bridges. If a vendor is absent from both, use `searchVendorPurchaseOrders(vendorName=...)`: it returns buyers with `stateCode` and `buyerType` and dated records (`date` = purchase or contract start, `untilDate`), independent of the target market. It has no state filter and is token-bounded (`limit` ≤ 80 records; `truncated=true` drops older buyers); narrow with `productName`, `alias`, `lookbackYears`, `limit`, and disclose when results may be incomplete.

## Joining presence and Top Buyers

Presence rows know the vendor and the contract; Top Buyers rows know the buyer's state, type and score. Any question that mixes the two (a vendor in a state, partners at top accounts, contracts in a region) is a two-step read:

1. Start on the side with the tighter filter — usually the presence bridge (`Vendor`, `Vendor Presence Equals true`) — and page fully, collecting `buyerId`s.
2. Read the other side with `buyerId Any [≤ 20 ids]` plus its own terms, one page per chunk.

## Interpretation rules

- "Best", "top", "priority" → sort by Account Scoring Score; "hottest", "right now", "active" → filter Signal Band Hot/Warm, then score. State which one you used.
- Regions and territories are not on any bridge. Expand them to explicit state names, say which you used, and filter `Buyer State Name Any [...]`.
- Buyers outside the target market have no row; report "not in the target market", not "score 0".
- Aggregations (counts by state, by vendor) are client-side: filter as tightly as possible, read the first page's `totalItems` and warn the user when it is large, then page fully and count. Tell the user the population is the target market only.
- `listBridgeRows` results can lag recent changes briefly; retry once if a just-scored row is missing.
- Cite evidence from `fit.longExplanation`, `signals.briefing`, presence `Vendor Presence Summary` / `Sources`, or `listRecentBuyerSignals`; do not infer relationships (e.g. that a partner will introduce the seller) beyond what the data states.
