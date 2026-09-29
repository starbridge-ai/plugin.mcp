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
5. Include every contact that has a *usable* email as a recipient — a real, unmasked address (`isMasked: false` with a non-null `email`); `isEnriched: false` is fine, drafting doesn't need enrichment. Only when a contact is masked (`isMasked: true`, email like `**********@domain`) is the address hidden until the contact is enriched via the credit-gated `enrichBuyerContact` tool: do NOT auto-enrich merely to draft; tell the user (cite the per-contact cost and remaining balance from `creditSpendHintsForContactActions` when present) and ask them to confirm enrichment first — see `contact-search` for the full flow. Draft with a generic greeting meanwhile.

## Prerequisites — Load Our Side of the Conversation

Call `getOrganizationBusinessDescription` with at least the `Positioning`, `CoreProblems`, `ProductsAndServices`, `CompetitiveDifferentiators` and `EmailGenerationPreferences` sections, in parallel with `contact-search`; they are independent. For the sign-off, `getCurrentUser` returns the user's `name`.

Ground our side in what it returns: do not invent product claims, customer outcomes, or differentiators it does not contain. The one exception is the user: facts they state about our own organization (customers, differentiators, products and services) can be used as given, because the business description can be stale. That exception covers our side only. When the user states a fact about the buyer that no tool returned (a funding decision, a budget figure, a board action), leave it out of the email and tell them you could not verify it.

## Ground the Personalization

Before drafting, make sure you actually have buyer context to personalize with — the opener and the pain/outcome must come from real data, not guesses:
- If real buyer context (priorities, momentum, outreach angles) isn't already in this conversation, activate `buyer-summary`; for recency-driven outreach, activate `buyer-signals` for a dated hook.
- Build the `Saw ...` line and the `Figured ...` line from a specific item in that data and reference it concretely.
- If neither returns usable context, fall back to title-based relevance and tell the user the email is only lightly personalized.

## Email Crafting Instructions

Write a first-touch outbound email the reader can take in at a glance. Assume they are one glance away from archiving.

1. Subject: 2 to 5 words, lowercase except proper nouns, no punctuation at the end, naming the specific topic or the buyer by its short common name (such as `course gaps at Maine Maritime`, not `Maine Maritime Academy`). No questions, no "quick question".
2. 30 to 50 words above the sign-off, one sentence per paragraph, blank line between paragraphs, in this order:
   - Greeting on its own line.
   - `Saw ...`: one specific, true observation about the prospect from the signal, research, or account context. Never a generic opener like "I hope this finds you well"; never invented.
   - `Figured ...`: the challenge or question that observation likely creates for them.
   - `Thought it'd be worth putting [seller company] on your radar. Brief synopsis below.`
   - `Happy to share more if that'd be helpful.`
   - Sign-off on its own line as `-` followed by the user's first name when available; otherwise leave a placeholder.
3. Below the sign-off, 2 to 4 one-line bullets starting with `- ` (never `*`): what the seller does for this prospect as concrete outcomes grounded in the organization's business description, plus a similar-customer proof point only when the business description supplies it.
4. Does not ask for a meeting, call, or time. The ask is the offer to share more.
5. Uses a plain, direct, human tone: contractions are fine, no exclamation points, no filler or AI-tell words ("leverage", "seamless", "streamline", "unlock", "empower", "robust", "innovative", "solution", "I'd love to", "navigate", "landscape", "elevate", "game-changer").
6. NEVER uses em dashes or en dashes in the email; hyphen-minus only.
7. When mentioning competitors, keeps the tone soft. Frame as "saw you're using XYZ; what teams that use XYZ tell us is..."
8. Does not include direct quotes from meetings, transcripts, files, snippets, or signal evidence. Use those sources to understand the account, then paraphrase them into a general pain point or business outcome. Do not put quotation marks around source language in the subject or body.
9. Uses facts and numbers only from the business description, the buyer data above, or what the user states about our own organization.

When `EmailGenerationPreferences` is configured, follow it for tone, length, and structure. It overrides the numbered rules above, including the first-email shape and word count, on those dimensions. The user's explicit request for this email wins over both. Neither overrides: no em dashes, no generic opener, no invented facts, no direct quotes from source material, and the output format below.

## Output Format

Before the draft, name the recipients you chose and why each one matters for the user's goal, in one or two sentences: role, seniority, decision-making fit, department ownership, a relevant signal, or likely influence on the purchase. Keep any other commentary equally brief; the email itself should be the clean, final thing the user reads.

Then present the draft as a readable email:

- **To:** — list each recipient as `Name, Title <email>`, one per line or comma-separated. Include every key contact that has an email address; omit any without one, and note web-sourced recipients as unverified. If no verified contact with an email was found, write `(none found, fill in manually)` and use a generic greeting in the body.
- **Subject:** — the subject line.
- The email **body** as plain text below, ready to copy-paste.

Example with found contacts:

**To:** John Smith, Chief Information Officer <john.smith@example.edu>; Jane Doe, VP of Operations <jane.doe@example.edu> (unverified)
**Subject:** course scheduling at Example State

Hi John,

Saw Example State is adding 40 course sections next semester.

Figured that puts real pressure on room and faculty scheduling.

Thought it'd be worth putting Acme on your radar. Brief synopsis below.

Happy to share more if that'd be helpful.

-Sam

- Builds conflict-free schedules across departments in days, not weeks
- Flags room shortfalls before registration opens

Example with no contact found:

**To:** (none found, fill in manually)
**Subject:** after-school programs at Example District

Hi there,

Saw your district is expanding after-school programs next year...
