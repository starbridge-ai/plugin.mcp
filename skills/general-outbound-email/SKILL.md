---
name: general-outbound-email
user-invocable: false
description: "Draft a personalized, concise B2B cold-outbound email for a BDR, grounded in real buyer and contact data."
when_to_use: "Use whenever the user asks to write, draft, generate, compose, or rewrite outbound email or cold outreach to a buyer. Example triggers — 'draft an email to X', 'write a cold email to the CIO', 'generate outreach for this district', 'compose a follow-up to them'. Activate contact-search first to find recipients, and buyer-identification before that if the buyer id is unknown."
---

# Outbound Email Generation

Draft a concise, compelling cold outreach email for a Business Development Representative (BDR). The email must be personalized to the buyer and grounded in real recipient data.

## Prerequisites — Find the Recipients

Before drafting the email you MUST identify who it should be sent to:
1. If you have NOT already activated `contact-search` in this conversation, activate it now and search for contacts at the target buyer.
   - If a particular contact was named, prioritize that contact. If the search fails to surface them, treat the recipient as not found.
2. Address ALL key contacts that have an email address. Do not limit to a single recipient — every contact returned with a valid email should be included as a recipient.
3. If a particular recipient was asked for by the user, use their contact details to personalize the greeting and body if that contact has an email.
4. If contact search returns no results or no contacts have an email address, leave the recipient list empty and use a generic greeting ("Hi there" or "Hi [Title]"). Tell the user you could not find a verified contact and they should fill in the recipient manually.
5. Include every contact that has a *usable* email as a recipient — a real, unmasked address (`isUnlocked: true` with a non-null `email`); `isEnriched: false` is fine, drafting doesn't need enrichment. Only when a contact's email is masked (e.g. `**********@domain`) or null — `isUnlocked: false` — is the address hidden until the contact is enriched via the credit-gated `enrichBuyerContact` tool: do NOT auto-enrich merely to draft; tell the user (cite the per-contact cost and remaining balance from `creditSpendHintsForContactActions` when present) and ask them to confirm enrichment first — see `contact-search` for the full flow. Draft with a generic greeting meanwhile.

## Ground the Personalization

Before drafting, make sure you actually have buyer context to personalize with — the opener and the pain/outcome must come from real data, not guesses:
- If real buyer context (priorities, momentum, outreach angles) isn't already in this conversation, activate `buyer-summary`; for recency-driven outreach, activate `buyer-signals` for a dated hook.
- Build the opening line and the pain/outcome from a specific item in that data and reference it concretely.
- If neither returns usable context, fall back to title-based relevance and tell the user the email is only lightly personalized.

## Email Crafting Instructions

Write a short outbound email (under 150 words) that:
1. Opens with a personalized first line that shows you've done your homework — do NOT use a generic opener like "I hope this finds you well"
2. Leverages a specific pain or business outcome the prospect likely cares about, grounded in the buyer context surfaced by `buyer-summary` / `buyer-signals` (or, if unavailable, the recipient's role)
3. Introduces your solution in one sentence, leading with value — not features
4. Closes with a low-friction CTA that feels like an easy "yes"
5. Uses the user's full name in the signature when available; otherwise leave a placeholder
6. Uses a conversational, human tone — no buzzwords, no fluff
7. NEVER uses em dashes in the email
8. When mentioning competitors, keeps the tone soft. Frame as "saw you're using XYZ — what teams that use XYZ tell us is..."

## Output Format

Present the draft as a readable email:

- **To:** — list each recipient as `Name, Title <email>`, one per line or comma-separated. Include every key contact that has an email address; omit any without one, and note web-sourced recipients as unverified. If no verified contact with an email was found, write `(none found — fill in manually)` and use a generic greeting in the body.
- **Subject:** — the subject line.
- The email **body** as plain text below, ready to copy-paste.

Keep any commentary brief and place it before the draft; the email itself should be the clean, final thing the user reads.

Example with found contacts:

**To:** John Smith, Chief Information Officer <john.smith@example.edu>; Jane Doe, VP of Operations <jane.doe@example.edu> (unverified)
**Subject:** Streamlining your campus scheduling

Hi John,

I noticed your team is expanding course offerings next semester...

Best,
[Your name]

Example with no contact found:

**To:** (none found — fill in manually)
**Subject:** Quick question about scheduling

Hi there,

I noticed your district is expanding...
