---
name: contact-search
user-invocable: false
description: "Find people at a buyer institution — staff, executives, board members, department heads — from Starbridge's verified contact database, with an optional public-web fallback. Also looks up specific people by known email address."
when_to_use: "Use for any 'who' or 'find contacts' question — whether the user names a role or a specific person — and as the required step before drafting outbound email. Also use when the user hands you email addresses to check, dedupe, or enrich. Example triggers — 'who is the CIO at X', 'find me contacts at this district', 'who runs IT / procurement / the board', 'get the superintendent', 'who should I reach out to', 'does Jason Klein work here', 'find Jane Doe at this district', 'what is Dr. Patel's email', 'do we have a verified contact for jklein@district.org', 'enrich this list of emails'. Run buyer-identification first if the buyer id is unknown."
---

# Contact Search

Finds contacts (people) associated with a buyer institution. Searches Starbridge's verified contact database, which contains staff, executives, board members, and department heads.

## When to Use
- Questions about who works at an institution
- Finding decision-makers, executives, or specific roles
- Looking up department heads or board members
- Any "who is..." or "find me contacts at..." query
- Confirming or enriching contacts you already have email addresses for
- As a prerequisite for drafting outbound email

## Tools

### `searchBuyerContacts`
Searches the Starbridge-verified contact database. Returns contacts with verified information including name, title, department, email, and phone. Supports three mutually exclusive lookup modes — pass exactly one:
- `include` (one or more canonical job titles, optionally narrowed with `exclude`). Pass titles as distinct canonical roles — not synonyms or abbreviations of the same role.
- `name` — a specific person's name (partial names are fine).
- `emails` — one or more known email addresses (up to 512). Addresses match case-insensitively; a malformed address is rejected rather than silently dropped. This is the only mode that returns inactive contacts by design: it matches any contact ever verified for that address at this institution, even if they've since left it, since finding them by email already proves Starbridge validated them in the past. Use this when the user supplies email addresses directly (e.g., checking a list, deduping, or enriching contacts they already sourced elsewhere) rather than asking "who is at X".

When no verified contacts match the request, the response may include `recommendedNextSteps`; surface those snippets to the user instead of deriving role-specific guidance yourself.

### `readUserInterfacePreferences`
Checks the current user's interface preferences. Use before prompting for an unlock when locked contacts are present. If `autoUnlockOnMCP` is true, the user has already consented to automatic contact unlocking during MCP sessions.

### `changeUserInterfacePreferences`
Stores or revokes the current user's interface preferences. Only set `autoUnlockOnMCP: true` after the user explicitly agrees that future MCP contact unlocks may happen without per-unlock confirmation and understands that each unlock consumes credits. Set `autoUnlockOnMCP: false` when the user asks to turn auto-unlock off.

### `enrichBuyerContact`
Enriches one or more contacts for the current organization. Enrichment is what makes a contact usable for further actions (CRM sync, sequences) and, when the organization masks contact details, reveals the masked fields. Use only after the user has invoked an unlock or enrichment action for one or more contacts. Consumes credits per newly enriched contact; already-enriched contacts are returned at no cost. If `readUserInterfacePreferences` returns `autoUnlockOnMCP: true`, execute as soon as the user invokes it. If auto-unlock is not enabled, ask for explicit consent before proceeding. If the user wants to enrich 20 or more contacts in one action, ignore `autoUnlockOnMCP` and ask for explicit consent before proceeding.

Work from `isEnriched` on each `searchBuyerContacts` result: `isEnriched: false` means the contact is not yet enriched for this organization and cannot be used for actions (CRM sync, sequences) until `enrichBuyerContact` succeeds. `isUnlocked` only tells the presentation layer whether details are masked; pass it through and do not base decisions on it.
`creditSpendHintsForContactActions.actionInContext` (`Unlock` or `Enrichment`) says which wording to use in this organization; use it whenever it is present. `canSpendFreely: true` means no credits are consumed; otherwise `costPerContact` is the per-contact cost.

### `runBuyerWebResearch`
Buyer-scoped public web search. Do not proactively offer this when verified contacts are unavailable. Use only when the user independently asks to search public web sources after being told no verified contacts matched their query.

## Workflow
1. Ensure the buyer has been identified first via `buyer-identification`
2. Search using `searchBuyerContacts` — pass `emails` when the user gave you specific addresses to look up, `name` when they named an individual, or one or more canonical job titles in `include` when they're asking by role. Never combine these.
3. If no verified contacts are returned, do not offer to pull other leadership contacts or search public web sources. Tell the user no verified contacts matched their request, then surface the `recommendedNextSteps` snippets from the `searchBuyerContacts` response as the call to action. Treat `recommendedNextSteps` as authoritative for this empty-search response. If `recommendedNextSteps` is empty or absent, use a conservative fallback:
   > "I didn't find verified contacts that match this request. Contact your Starbridge admin to enrich more verified contacts with Starbridge."
4. Present contacts using the structured output format below. When any contacts are not enriched (`isEnriched: false`), call `readUserInterfacePreferences` before asking for unlock or enrichment confirmation.
5. If the user invokes an unlock action and `autoUnlockOnMCP` is true, call `enrichBuyerContact` for the relevant locked contacts, then re-present the updated contacts. Tell the user that auto-unlock was already enabled and include the credits spent or remaining credit context from the unlock response when available. If the user invokes an unlock action for 20 or more contacts at once, ignore `autoUnlockOnMCP` and ask for explicit consent before calling `enrichBuyerContact`.
6. If not-enriched contacts (`isEnriched: false`) are present and `autoUnlockOnMCP` is false, frame the unlock or enrichment option as an encouraged action (either action makes the contact usable for CRM sync or sequences; `Unlock` additionally reveals masked details) using the `creditSpendHintsForContactActions` from the `searchBuyerContacts` response. Before stating the cost, briefly indicate which fields are available for each contact to be enriched (e.g. "has email and phone", "has email only") so the user knows what they'll get before spending credits. If `creditSpendHintsForContactActions` is unavailable, still recommend unlocking or enriching the relevant contacts, but do not quote credit balances. In the templates below, `[Action]` is "Unlock" when `actionInContext` is `Unlock` and "Enrich" when it is `Enrichment`; if `actionInContext` is absent, use "Enrich". `canSpendFreely: true` means no credits are consumed and `costPerContact` is the per-contact cost:
   - **No personal limit, org has a credit balance** (`hasPersonalLimit: false`, `orgCreditsRemaining` is not null):
     > "[Action] for [cost] credit(s) — your team has [orgCreditsRemaining] credits remaining, and you're free to enrich as many contacts as you need."
   - **No personal limit, org is on an unlimited plan** (`hasPersonalLimit: false`, `orgCreditsRemaining` is null):
     > "[Action] for [cost] credit(s) — you're free to enrich as many contacts as you need."
   - **Per-user limit set** (`hasPersonalLimit: true`):
     > "[Action] for [cost] credit(s) — your admin has set a personal limit and you have [personalCreditsRemaining] remaining. [Action] as many contacts as you need. You can always ask your admin for a top-up."
   - **Credit hints unavailable** (`creditSpendHintsForContactActions` is null or incomplete):
     > "I found verified contacts that look useful for this request but aren't enriched yet. Enriching them makes them available for CRM sync and sequences and, if details are masked, reveals the available email and/or phone fields. Would you like me to enrich them?"
7. When asking for unlock confirmation, offer to remember the user's choice for future MCP contact unlocks only if it is natural in the conversation. If the user explicitly agrees to remember the choice, call `changeUserInterfacePreferences` with `autoUnlockOnMCP: true` before or alongside unlock. If the user asks to stop automatic unlocks, call `changeUserInterfacePreferences` with `autoUnlockOnMCP: false`.
8. Once the user confirms they want to unlock, call `enrichBuyerContact` and re-present the updated contacts.

## Output Format
The `searchBuyerContacts` response includes `contacts`, `creditSpendHintsForContactActions`, and `recommendedNextSteps`.
Each contact includes: `id`, `firstName`, `lastName`, `middleName`, `salutation`, `title`, `email`, `phone`, `isActive`, `isUnlocked`, `isEnriched`, `organizationAttributes`. Base all decisions on `isEnriched`; `isUnlocked` is presentation-only (whether details are masked) and is passed through as-is.
Present contacts in a clean, readable format — use name, title, and contact details as the primary information. Omit null or empty fields from the display.
For an `emails` lookup, `isActive: false` is an expected, common result — not a data quality issue. It means Starbridge verified that contact at this institution at some point but they are no longer current; call this out to the user (e.g., "no longer active at this institution") rather than treating it as stale or unreliable data.

## Important
- Never fabricate or guess contact information
- When verified contact search returns no matches, do not suggest other leadership contacts or public web search unless the user explicitly asks for that next
- Never enable `autoUnlockOnMCP` unless the user clearly consents to future automatic contact unlocks and understands that unlocks spend credits
- All contacts returned by `searchBuyerContacts` are Starbridge-verified and reliable, regardless of `isActive`; web contacts may be outdated — always indicate the source. `isActive: false` means no longer current at the institution (expected and common for `emails` lookups), not that the data itself is stale
- Users can create a contact verification bridge in the application if they need ongoing contact monitoring
