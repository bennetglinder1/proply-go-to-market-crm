# Entity Resolution

Every GTM tool assigns its own identifier to the same person. RB2B identifies a site visitor by IP and email. Instantly tracks them by sequence contact ID. LinkedIn sends a profile URL. Fireflies logs them by display name in a meeting transcript. Google Calendar sends an attendee email with no display name. These identifiers have nothing in common — and yet they all describe the same human being in your pipeline.

Entity resolution is the continuous, real-time problem of matching incoming signal records to the right canonical contact. We built a tiered resolution waterfall that handles this across every integration Proply connects to, under latency constraints that don't allow for batch processing or LLM inference on every record.

![Multiple sources — LinkedIn, Gmail, Instantly, Notion, RB2B — all pointing to the same contact with question marks](../assets/entity-resolution-sources.png)

## One Person, Many Identifiers

![Same person identified differently across LinkedIn (URL), Gmail (personal email), Instantly (work email), RB2B (domain) — no shared key](../assets/entity-resolution-identifiers.png)

No universal identifier exists across GTM tools. Email is the closest — but it is only present in 60–70% of incoming signals. A Fireflies transcript gives you a participant name with no email. A LinkedIn connection event gives you a profile URL with no email. An RB2B visit gives you an email but a different name format than the same person's record in HubSpot. And the same person often has multiple emails across different tools — personal Gmail in one, work address in another.

Getting resolution wrong in either direction is expensive. A missed match creates a duplicate — your agent treats a warm lead as a cold unknown and starts from scratch. A wrong merge corrupts two relationships into one — your agent acts on someone else's history. Resolution errors compound: one bad match in January corrupts every signal from that source for the life of that contact.

The other pressure is latency. Webhooks fire in real time. You have milliseconds to resolve identity before the signal is stored. Queueing for batch processing means losing the signal or storing it incorrectly.

## The Resolution Waterfall

We resolve every incoming signal through a tiered waterfall, stopping at the first successful match. Each step is ordered by speed and certainty — the fast path covers the majority of signals, the slow path handles the hard cases.



**External platform ID** — every integration assigns its own stable identifier: , , a sequence contact ID from Instantly. On first encounter we store it alongside the canonical contact. Every subsequent event from that platform matches instantly. This covers roughly 70–80% of signals from active integrations.

**Email (normalized)** — when an external ID isn't present, email is ground truth. Normalization matters: lowercase everything, strip display name wrappers ( → ), handle  aliases consistently. Un-normalized matching causes silent misses.

**LinkedIn URL (normalized)** — a stable unique identifier per person, but it arrives in multiple formats: with and without , with trailing slashes, as a full URL or just the path. We normalize all forms to the slug and match against a normalized index.

**Email local-part inference** — for integrations like Google Calendar where the API provides an attendee email but no display name, we parse the email local-part and attempt name matching against contacts with no email on file.  →  → match against .  →  → match against . Single match — auto-resolve, write the discovered email back to the contact record permanently. The self-healing write means the next event from that email hits step two instantly.

When a local-part produces multiple candidates — three contacts named Spencer — we tiebreak using the last 14 days of . Any candidate with a meeting-intent signal in their history (book, schedule, call, calendly, zoom) scores. Exactly one scorer — resolve that contact. Tied or zero scorers — reject. This covers the real case: if you just exchanged LinkedIn messages about booking a call and then a calendar event arrives with a matching  email, the meeting-intent signal in Spencer's activity log disambiguates him from every other Spencer in the workspace.

**Fuzzy name + company** — for signals that carry a name and company but no email or external ID (primarily meeting transcripts from Fireflies), we compute a composite similarity score combining token sort ratio, phonetic encoding, and edit distance for names, plus token overlap for company names. Above 0.90 — auto-merge. Between 0.70–0.90 — flag for human review. Below — new contact or reject depending on source.

At every step, the identifiers used to find a match get patched back onto the contact record. The first time someone is matched by name, their external ID is stored — the next event from that source hits step one instantly.

## Source Trust and Create / Reject

**[VISUAL: Three source levels — L1 prospecting tools (create on no match) / L2 communication (reject) / L3 meeting tools (reject) — with example integrations at each level]**

Not every unmatched signal should create a new contact. The decision depends on where the signal came from.

**Level 1 — prospecting sources** (RB2B, Instantly, LinkedIn, Apollo, HubSpot): create on no match. These represent intentional outbound activity. If someone visited your site and isn't in the system, they're a real lead.

**Level 2 — communication sources** (Gmail): reject on no match. Anyone who has ever emailed you would become a contact. The noise ratio is too high.

**Level 3 — meeting sources** (Fireflies, Calendly, Fathom, Google Calendar): reject on no match. Meeting participants aren't necessarily prospects. If no match exists, the signal is discarded — .

Without this distinction, a single 20-person all-hands transcript creates 20 spurious contacts and fills your pipeline with noise.

Resolution is also idempotent by design. Webhook providers retry failed deliveries — the same signal might arrive 2–5 times. A unique constraint on  in the activity log means processing the same event twice produces the same result as processing it once.

## Principles

**Email is ground truth, but it's not always present.** Design for the 30–40% of signals where it's missing. The waterfall has to handle those cases without degrading to create-on-every-miss.

**Store every identifier you encounter.** The first time a contact passes through any tier, patch that tier's identifier back onto the record. Every identifier stored is one fewer lookup needed next time. This applies to discovered emails too — an email resolved from a local-part match gets written back immediately.

**Source trust determines create vs. reject.** A signal that doesn't match is not automatically a new contact — the right answer depends entirely on where it came from.

**When candidates tie, activity history disambiguates.** The same pipeline that stores relationship signals can break identity ties. A contact with a recent meeting-intent signal in their activity log is the right match for a calendar event with a matching first name — not because of inference, but because the signal already happened and was recorded. This only works when the activity log is granular and written in real time.

**Resolution errors compound.** A bad merge in a live stream corrupts every future signal from that source. The asymmetry in error cost — false negative creates a duplicate, false positive corrupts an identity — argues for a high merge threshold and a human review queue for uncertain cases.

**The hard cases are known.** Abbreviations ( vs ), nicknames ( vs ), and meeting participants with no email are the three cases the waterfall won't catch automatically. They're addressable — lookup tables, phonetic encoding, enrichment — but worth naming so they don't become silent failures.
