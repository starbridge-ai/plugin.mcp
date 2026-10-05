---
name: account-scoring
user-invocable: false
description: "Read and configure the organization's Account Scoring (the Top Buyers bridge) and the competitor and complementary-vendor presence bridges of its market — account scores, fit and signal bands, rankings of target accounts, which accounts a competitor or partner is present in, with contract values and dates, and what drives the score."
when_to_use: "Use whenever the user asks about account scores, fit or signal bands, best or hottest accounts, target-account rankings, ICP prioritization, which accounts a competitor or partner is present in, which buyers in a state or of a type use a vendor, a vendor's contract values or dates, a vendor's footprint by state, anything that ranks, filters, or aggregates buyers in their target market, or a request to change what drives the score (fit attributes, guidance, signal types) — even if they never say 'Top Buyers' or 'bridge'. For other bridges use bridges; for one institution's own data use buyer-summary, document-research, or buyer-attributes."
---

# Account Scoring (Top Buyers)

Each organization has at most one Top Buyers bridge, built on its total market (`listMarkets`, `isTam`). It has one row per in-market buyer, scored 0–100 by combining **Fit** (how well the buyer matches the ICP) with **Signals** (buying intent from the organization's other bridges). The market also owns two presence bridges, one row per buyer × configured vendor: competitors (`ProductCompetitorPresence`) and complementary vendors (`CompanionPresence`).

Customers never say "Top Buyers": use the names `listMarkets`, `getBridgeFull` and `getBuyerAccountContext` give (Account scoring, Total market, Competitors, Complements, and the score tier and band labels).

## One account: `getBuyerAccountContext`

For a question about a single buyer ("why is X a 72?", "is X in market?", "who's their incumbent?"), call `getBuyerAccountContext(buyerId)` instead of reading the bridge. It returns `accountScore` (0–100), `fitBand`, `signalsBand`, `inMarket`, and `scoringAttributes`, the fit columns with each value, guidance and the model's `explanation`. Add `includeVendorPresence=true` (the expensive section) for competitor and complementary vendors with contract dates and values. Ignore entries with `usedByBuyer=false`.

Sections are best effort, so a null score is not proof the account is unscored. For the signals narrative, read the row's Account Score cell or call `getBuyerSummaryV2`.

## Many accounts: the bridge

1. Find the bridge with `listBridgesFull(filterType=[TopBuyer])` or `listMarkets` (`bridgeSettings.topBuyerBridgeId`, plus the two presence bridge ids). If there is none, say Account Scoring is not set up; do not substitute another bridge.
2. Call `getBridgeFull(bridgeId, includeConfiguration=false)`. Filters, sorts and `columnIds` take `columnId` UUIDs only, never names. Record the ids of:
   - **Account Scoring Score**: the number. Rank on this column.
   - **Fit Band** and **Signal Band**: enums, hidden in the UI.
   - **Account Score**: `type=AccountScoring`, key `top-buyer:account-scoring`. Its value is `{score, fit{band, shortExplanation, longExplanation{summary, factors[{phaseName, explanation}]}}, signals{band, shortExplanation, briefing{overview, opportunities, accountContext}, sources}}`. Project it for explanations; never filter on it.
   - **Buyer State Name** and **Buyer Type**.
   - Link columns **Competitive vendors** and **Complementary vendors** (`linkSettings` points to the presence bridges).
   - MultiLink signal columns such as "RFP Signals": a count per row.
   - Customer-added columns, such as AI analysis, web agent or CRM lookups. They may answer the question directly.
3. `listBridgeRowsFull`:
   - **Always send `columnIds`.** Without it, every column is returned, including the hidden ones. Link columns are resolved only when projected (at most 3 `previewRows` plus `total`), so leave them out and `pageSize` can go to 100.
   - **Rank:** `sorts=[{column:<Account Scoring Score>, direction:DESC}]`.
   - **Hottest / active / right now:** filter Signal Band `Any ["Hot","Warm"]`, then rank.
   - **Geography and type:** use `buyerScope.terms`, the same fields as `searchBuyersByCriteria`. Examples: `{field:StateCode, operation:Any, value:["TX","OK"]}`, `{field:Type, operation:Any, value:["City","County"]}`, and population or enrollment ranges. Expand regions such as "the Southeast" into explicit states, and say which ones you used.
   - **Counts** ("how many Hot accounts in Florida?"): call `getBridgeRowStats` with the same `filters` and `buyerScope` and read `total`. Don't page. It takes no `query`, so express that constraint as a filter term.
   - **Shape:** rows are `{rowId, buyerId, name, columns}`, with `columns` keyed by column name. Each cell is `{value, status, processedAt}`.
   - **Paging:** increment `pageNumber` while it is below `totalPages`.
   - **Consistency:** the index can lag; retry once if a just-scored row is missing.

## Vendor presence

Read the presence bridge itself when a link cell's `total` exceeds 3 or when the question is about vendors:

- `getBridgeFull` for the column ids of **Vendor**, **Vendor Presence** (boolean), **Vendor Presence Summary** and **Estimated Annual Contract Value**. Also, when present, **Estimated Total Contract Value**, **Estimated Contract End Date**, **Estimated Contract Term** (a "start – end" range) and, on competitors only, **Products**.
- `listBridgeRowsFull` with `Vendor Presence Equals true`, plus `Vendor Any [...]` and `buyerScope` as needed.

Contract values, end dates and terms are estimates; say so, and keep the Summary's citation links. Citations ending in `…/opportunity/<id>` can be opened with `getOpportunityLineItems` or `viewFileContents`. There is no signing date.

Only vendors configured on the market appear (`listMarkets`, `bridgeSettings.competitorProducts` / `companionPresence`). For any other vendor, or for "most recent contract", use `searchVendorPurchaseOrders(vendorName, alias, productName)`:
- It covers every buyer, not just the market. Each record carries `stateCode`, `buyerType`, `date` and `untilDate`.
- It has no state filter, and results are capped at 80 records (`truncated=true` drops older buyers). Narrow the search and disclose when results may be incomplete.

**Score × vendor questions** ("hot accounts using Competitor X"): read the tighter side first and collect `buyerId`s. Then read the other bridge with `filters` `buyerId Any [...]` in chunks of at most 100, each chunk also carrying that bridge's own terms.

## Configuring the score

`updateAccountScoring` requires the Admin or Builder role.

1. `getBridgeFull(topBuyerBridgeId)`. The Account Score column's `accountScoringMetadata` is the current config:
   - `scoringConfig.fitColumns[{phaseId, includedInScoring, strongFitGuidance}]`, where `phaseId` is a `columnId`.
   - `fitInteractionPrompt`, `signalInteractionPrompt`, and `signalConfigs{<type>: {guidance}}`.
   - `signalBridgesByType{<type>: {id: <MultiLink columnId>, enabled}}`.
2. **Fit.** Any column with a value can be a fit column: a buyer attribute, AI analysis, web agent, CRM lookup, custom field, or the vendor Link columns. It cannot be the Account Score column or its children.
   - If the attribute the user describes is missing, add it with `addBridgeColumn` (`Attribute`, `AiAnalysis`, `WebAgent` or `VendorPresence`). Leave `autoRunOnRowAdd` on.
   - Fill it with `processBridgeRows`. This spends credits, so run `estimateProcessBridgeRows` first.
   - Write `strongFitGuidance` as a rubric for that column: "When does this indicate a strong buyer fit?" Use `fitInteractionPrompt` for how attributes trade off ("a low AI-adoption score is fine if no competitor is present").
3. **Signals.** Each type is scored from one MultiLink column on the Top Buyers bridge that links that type's signal bridges. Org setup usually creates "RFP / Meeting / Job Change / Custom Web / Contract Expiration Signals".
   - **Enroll a bridge:** add its id to that type's MultiLink with `updateBridgeColumn(linkedBridgeIds)`, sending the full list.
   - **New type:** `addBridgeColumn(kind=MultiLink, linkedBridgeIds, linkSettings.includedStatuses=["New","Actioned","Saved"])`, linking only bridges of that type.
   - **Valid types:** `RFP`, `Meeting`, `JobChange`, `Signal`, `ContractExpiration`, `Purchase` and `Conference`.
   - `guidance` says which signals of the type show real intent. `signalInteractionPrompt` says how types combine ("a relevant RFP alone makes an account hot").
4. **Update.** Send only what changes to `updateAccountScoring(bridgeId, ...)`:
   - Fit columns are merged by `columnId`; use `removeFitColumnIds` to drop them, or `includedInScoring=false` to keep a column without counting it.
   - Signals are merged by `type` (`{type, columnId, enabled, guidance}`); use `removeSignalTypes` to drop them.
   - An empty string clears a prompt or a guidance. Omitting a field keeps it.
   - Invalid references, including a MultiLink that mixes bridge types (`SignalColumnTypeMismatch`), are reported together in one error. Relay any `warnings` (`NoScorableInputs`, `FitColumnsNotAutoRun`, `DeletedColumnsRemoved`, `ScoresNotRecomputed`).
5. **Share.** `shareAccountScores` / `shareEnrichmentColumns` on the same tool set who sees the scores and enrichment columns; `getBridgeFull` reports them as `configuration.accountScoringSharing`. Confirm with the user first: sharing scores shows them to every member.
6. **Rescore.** Changes only mark scores Outdated; the daily run refreshes them. To refresh now, call `processBridgeRows(columnIds=[Account Score], filters...)`. It defaults to `Rerun`, with `limit` at most 200; target the accounts the user cares about first. Then poll `getBridgeRowStats` with the `columnIds` and `rowIds` it returns until those cells leave `Queued` / `Processing`.

Outside this tool set:
- The market's buyer scope (`saveMarket`) and its competitor or complement vendors (`setMarketPresenceBridge`) are Admin-only, re-run the market's bridges and spend credits.
- `migrateBridgeToTopBuyers` (Admin-only) converts an old `Buyer` account-scoring bridge.

## Interpretation rules

- "Best", "top" or "priority" means rank by score. "Hottest", "right now" or "active" means filter Signals Hot/Warm, then rank. A score tier ("Burning accounts", "everything Hot and up") means filtering Account Scoring Score at that tier's threshold, as `getBridgeFull` lists them. State which one you used.
- The population is the total market only. Say so when you count or rank.
- Cite evidence from `fit.longExplanation`, `signals.briefing`, `scoringAttributes[].explanation` or the Vendor Presence Summary. Do not infer relationships beyond what the data states.
