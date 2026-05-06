# GTM Entity Resolution

Every GTM tool assigns its own identifier to the same person. RB2B identifies a site visitor by IP and email. Instantly tracks them by sequence contact ID. LinkedIn sends a profile URL. Fireflies logs them by display name in a meeting transcript. These identifiers have nothing in common — and yet they all describe the same human being in your pipeline.

Entity resolution is the continuous, real-time problem of matching incoming signal records to the right canonical contact. We built a tiered resolution waterfall that handles this across every integration Proply connects to, under latency constraints that don't allow for batch processing or LLM inference on every record.

**[VISUAL: Architecture — multiple sources (RB2B, Instantly, LinkedIn, Fireflies, HubSpot) all feeding into a single resolution layer before writing to contact_activity_log]**

## The Identifier Problem

**[VISUAL: Same person across four sources — four different identifiers, none of them matching]**

The core difficulty is that no universal identifier exists across GTM tools. Email is the closest — but it's present in only 60–70% of incoming signals. A Fireflies transcript gives you a participant name with no email. A LinkedIn connection event gives you a profile URL with no email. An RB2B visit gives you an email but a different name format than the same person's record in HubSpot.

Getting resolution wrong in either direction is expensive. A missed match creates a duplicate — your agent treats a warm lead as a cold unknown and starts from scratch. A wrong merge corrupts two relationships into one — your agent acts on someone else's history. Resolution errors compound: one bad match in January corrupts every signal from that source for the life of that contact.

The other pressure is latency. Webhooks fire in real time. You have milliseconds to resolve identity before the signal is stored. Queueing for batch processing means losing the signal or storing it incorrectly.

## The Resolution Waterfall

**[VISUAL: Tiered waterfall — External ID → Email → LinkedIn URL → Fuzzy name+company → Create or reject]**

We resolve every incoming signal through a 4-step waterfall, stopping at the first successful match. Each tier is ordered by speed and certainty — the fast path covers the majority of signals, the slow path handles the hard cases.

**Step 1 — External platform ID.** Every integration assigns its own stable identifier: `rb2b_id`, `hubspot_id`, a sequence contact ID from Instantly. On first encounter we store it alongside the canonical contact. On every subsequent event from that platform, it matches instantly — single index lookup, deterministic. This covers roughly 70–80% of signals from active integrations.

**Step 2 — Email (normalized).** When an external ID isn't present, email is ground truth. Normalization matters more than it looks: lowercase everything, strip display name wrappers (`"Ben Carter <ben@acme.com>"` → `ben@acme.com`), handle `+` aliases consistently. Un-normalized matching causes silent misses — `Ben@Acme.com` and `ben@acme.com` are the same address, a case-sensitive lookup creates a duplicate.

**Step 3 — LinkedIn URL (normalized).** LinkedIn URL is a stable unique identifier per person, but it arrives in multiple formats: with and without `www`, with trailing slashes, as a full URL or just the path. We normalize all formats to the slug (`bencarter`) and match against a normalized index.

**Step 4 — Fuzzy name + company.** For signals that carry a name and company but no email or external ID — primarily meeting transcripts from Fireflies — we compute a composite similarity score:

```js
score = w1 * name_similarity(incoming, candidate)
      + w2 * company_similarity(incoming, candidate)
      + w3 * domain_match(incoming_domain, candidate_domain)
```

Where `name_similarity` combines token sort ratio (handles `"Carter Ben"` vs `"Ben Carter"`), phonetic encoding (handles `"Ben"` vs `"Bennet"`), and edit distance for typos. Above 0.90 — auto-merge. Between 0.70–0.90 — flag for review. Below 0.70 — treat as new or reject depending on source.

## Source Trust and Create / Reject

Not every unmatched signal should create a new contact. The decision depends on where the signal came from.

We split sources into three levels:

**Level 1 — prospecting sources** (RB2B, Instantly, LinkedIn, Apollo, HubSpot): create on no match. These represent intentional outbound activity. If someone visited your site and isn't in the system, they're a real lead.

**Level 2 — communication sources** (Gmail): reject on no match. Anyone who has ever emailed you would become a contact. The noise ratio is too high.

**Level 3 — meeting sources** (Fireflies, Calendly, Fathom): reject on no match. Meeting participants aren't necessarily prospects. If no match exists, the signal is discarded — `createIfMissing: false`.

Without this two-class treatment, a single 20-person all-hands transcript creates 20 spurious contacts. One company all-hands meeting fills your pipeline with noise.

Resolution is also idempotent by design. Webhook providers retry failed deliveries — the same signal might arrive 2–5 times. A unique constraint on `(source, external_id)` in the activity log means processing the same event twice produces the same result as processing it once.

## Principles

**Email is ground truth, but it's not always present.** Design your resolution system to work without it. Email covers 60–70% of signals; the waterfall has to handle the other 30–40% correctly.

**Store every identifier you see.** The first time a contact passes through any tier, patch that tier's identifier back onto the contact record. Every identifier stored is one fewer lookup needed next time.

**Source trust determines create vs. reject.** A signal that doesn't match is not automatically a new contact. The right answer depends entirely on where the signal came from.

**Resolution errors compound.** Unlike a static dataset where a bad match affects one record, a bad merge in a live stream corrupts every future signal from that source. The asymmetry in error cost — false negative creates a duplicate, false positive corrupts an identity — argues for a high merge threshold and a human review queue for uncertain cases.

**The hard cases are known.** Abbreviations (`IBM` vs `International Business Machines`), nicknames (`Ben` vs `Bennet`), and meeting participants with no email are the three cases the waterfall won't catch automatically. They're addressable — lookup tables for abbreviations, phonetic encoding for nicknames, enrichment for meeting participants — but worth naming explicitly so they don't become silent failures.
