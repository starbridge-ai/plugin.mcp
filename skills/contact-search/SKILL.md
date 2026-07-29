---
name: contact-search
user-invocable: false
description: "Find people at a buyer institution — staff, executives, board members, department heads — from Starbridge's verified contact database, with an optional public-web fallback."
when_to_use: "Use for any 'who' or 'find contacts' question — whether the user names a role or a specific person — and as the required step before drafting outbound email. Example triggers — 'who is the CIO at X', 'find me contacts at this district', 'who runs IT / procurement / the board', 'get the superintendent', 'who should I reach out to', 'does Jason Klein work here', 'find Jane Doe at this district', 'what is Dr. Patel's email'. Run buyer-identification first if the buyer id is unknown."
---

# Contact Search

Finds contacts (people) associated with a buyer institution. Searches Starbridge's verified contact database, which contains staff, executives, board members, and department heads. Falls back to public web research when no verified contact is available.

## When to Use
- Questions about who works at an institution
- Finding decision-makers, executives, or specific roles
- Looking up department heads or board members
- Any "who is..." or "find me contacts at..." query
- As a prerequisite for drafting outbound email

## Tools

### `searchBuyerContacts`
Searches the Starbridge-verified contact database. Returns contacts with verified information including name, title, department, email, and phone. Search one of two ways:
- **By person name** — pass `name` (e.g. `Jason Klein`) when the user names a specific individual rather than a role. You don't need their title first. A partial name is enough: a first name alone, a surname alone, or both, and unknown middle names don't matter. Pass the name as the user gave it — don't expand or guess at a fuller version. Closest matches come back first; when several people share the name, disambiguate with the user rather than assuming the first is the one they meant.
- **By job title** — pass one or more canonical job titles via `include` (and optionally `exclude`) when the user asks for a role. Pass titles as distinct canonical roles — not synonyms or abbreviations of the same role.

Provide exactly one of `name` or `include` — supplying both is rejected. If the user named neither a person nor a role, infer likely decision-maker titles (e.g. Superintendent, Chief Information Officer) or ask which roles they want before searching. When the `contacts` list comes back empty, the response also carries role-aware next-step suggestions — surface those rather than silently reporting nothing.

### `unlockBuyerContact` (credit-spending — confirm first)
Available for organizations with contact gating enabled; unlocks a locked contact's full details and **spends credits**. The `searchBuyerContacts` response marks whether each contact is unlocked and includes `unlockCreditSpendHints` (per-contact cost and remaining balance). Unlocking MUST be user-initiated: tell the user the contact is locked, state the cost and remaining balance from `unlockCreditSpendHints`, and get explicit confirmation before calling — even when unlocking is free. Pass the contact `id` from the search response.

### `runBuyerWebResearch`
Buyer-scoped public web search. Use only when verified contacts are unavailable AND the user has approved searching public sources.

## Workflow
1. Ensure the buyer has been identified first via `buyer-identification`
2. Search using `searchBuyerContacts`: when the user named a specific person, pass their name in `name`; when the user asked for a role, pass one or more canonical job titles in `include`. Never pass both in one call — the tool rejects that; if the user gave a name *and* a role, search by `name` and check the titles that come back. If neither was named, infer likely titles or ask first
3. Present contacts using the **Output Format** below (a readable table)
4. If a contact the user wants is locked, follow the `unlockBuyerContact` confirmation flow above before unlocking — never unlock without explicit user confirmation
5. If the `contacts` list is empty, surface the response's role-aware next-step suggestions; do not silently report nothing. Only then, ask before using `runBuyerWebResearch`:
   > "I didn't find verified contacts for this role. Would you like me to search public web sources? Note that web results may be less reliable."

## Output Format

Present contacts as a readable Markdown table so they're easy to scan — one row per contact, with columns **Name**, **Title**, **Department**, **Email**, **Phone**, and **Verified**.

- Use the contact's full name (include salutation and middle name when available).
- Show `—` for any missing value rather than leaving a cell blank.
- In the **Verified** column, mark Starbridge-verified contacts `✓ Verified` and web-sourced contacts `Unverified (web)`.
- For a locked contact, show `🔒 locked` in the Email and Phone cells and follow the `unlockBuyerContact` confirmation flow before revealing details.
- Lead in with a short sentence (who you found, for which roles), and add a brief follow-up after the table if useful. Don't fabricate or pad missing fields.

Example:

| Name | Title | Department | Email | Phone | Verified |
|------|-------|------------|-------|-------|----------|
| Ms. Emily Johnson | Director of Institutional Research | Institutional Research | emily.johnson@example.edu | +1-201-555-0142 | ✓ Verified |
| Michael Rivera | Associate Vice President of Academic Affairs | Academic Affairs | michael.rivera@example.edu | — | Unverified (web) |

## Verified vs. Unverified Contacts
- **Starbridge-verified contacts**: From `searchBuyerContacts`. Reliable and up-to-date. Mark them `✓ Verified` in the table.
- **Web contacts**: From `runBuyerWebResearch`. May be outdated or inaccurate. Mark them `Unverified (web)` in the table.

## Important
- Never fabricate or guess contact information
- Always distinguish between verified and unverified sources via the `isVerified` field
- Users can create a contact verification bridge in the application if they need ongoing contact monitoring
