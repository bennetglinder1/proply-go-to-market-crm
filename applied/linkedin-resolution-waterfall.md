# LinkedIn Contact Resolution — The Unipile Waterfall

We built a live LinkedIn signal pipeline for Proply using Unipile as the middleware layer. The goal: every message sent, message received, and new connection automatically appears on the right contact in Proply — with the correct timestamp, no duplicates, no manual work. This is the system we landed on, and the two structural problems that took the longest to solve.

## The Setup

**[VISUAL: Architecture diagram — Unipile → Render backend → Supabase → Proply contact timeline]**

Unipile connects to your LinkedIn account and fires webhook events to our Express backend (hosted on Render) every time something happens: a message arrives, you send one, a new connection is made. Our backend receives those events, resolves which contact they belong to, and writes them to `contact_activity_log` in Supabase. From there, Proply surfaces them in the contact timeline and uses them to advance pipeline stage automatically.

Two event types are wired up: `message_received` (fires on every sent or received LinkedIn message) and `new_relation` (fires on new connections). We discard everything else — `message_read`, `message_delivered`, `message_reaction`, and a handful of other non-actionable events — before any processing touches them. These events contain a `message_id` field that looks identical to an actionable event, and leaving them in the pipeline poisons the dedup logic.

## The Resolution Problem

**[VISUAL: Two LinkedIn URL formats side by side — readable slug vs ACoAA member ID — showing they share nothing in common]**

The central challenge in this integration is identity. When a LinkedIn webhook arrives, you know the sender's name, their LinkedIn URL, and a LinkedIn member ID. When a contact exists in Proply, they might have been imported from a CSV with a human-readable URL, added manually, or created from a prior webhook. The question is: are these the same person?

LinkedIn makes this harder than it looks. The same person has two completely different URL formats depending on where you encounter them:

| Format | Example | Source |
|---|---|---|
| Readable slug | `/in/georgi-furnadzhiev` | CSV imports, manual entry, airtable |
| Member ID | `/in/ACoAADL-V-kBZ5Dd-k90zDpKft0ia3Qr36MFn1M` | Unipile message event payloads |

These strings have nothing in common. A standard `ilike '%/in/georgi-furnadzhiev%'` match against a stored URL of `/in/ACoAADL...` returns zero results. This isn't a Unipile quirk — it's how LinkedIn's infrastructure works. The readable slug is a vanity alias. The member ID is the permanent internal identifier. Before we understood this distinction, every webhook for a contact imported from CSV would fail URL matching and create a duplicate.

The second structural issue was how name matching degrades at scale. When contacts don't have a LinkedIn URL at all (common for contacts imported before integrations were connected), we fall back to matching by first and last name. Using Supabase's `.maybeSingle()` for this works fine with one record — but returns `null` when two or more records match. Once a duplicate exists in the database, every future name-match attempt finds `null` and falls through to creating a third contact. Duplicates compound.

## The Resolution Waterfall

**[VISUAL: 4-step waterfall diagram — member ID → URL slug → name → create — with self-healing patch arrows at each step]**

We resolved this with a 4-step identity waterfall that runs on every LinkedIn webhook event. Each step self-heals: when a contact is found, the identifiers that were used to find them get patched back onto the record. Over time, the system converges to matching everyone by member ID with no URL or name logic required.

```js
resolveLinkedInContact(supabase, workspaceId, { linkedinUrl, fullName, memberId })
```

**Step 1 — LinkedIn Member ID (permanent).** Query `contacts.linkedin_member_id = memberId`. The `ACoAA...` identifier is permanent — it never changes regardless of URL updates or profile edits. Once it's stored, this step is instant. If found, patches `linkedin_url` back if missing.

**Step 2 — LinkedIn URL slug match.** `ilike contacts.linkedin_url '%/in/<slug>%'` catches contacts imported with a readable slug URL. Patches `linkedin_member_id` back so step 1 works next time.

**Step 3 — First + last name match.** Safety net for contacts with no LinkedIn data. Uses an array query — not `.maybeSingle()` — so existing duplicates don't return null. Prefers the record with an existing `linkedin_url` when multiple name matches exist. Patches both `linkedin_url` and `linkedin_member_id` back.

**Step 4 — Create.** Creates a new contact only as a last resort. Immediately patches `linkedin_member_id` onto the new record so step 1 works the next time this person triggers a webhook.

The member ID itself comes from Unipile's `sender.attendee_provider_id` on message events and `user_public_identifier` on connection events. We added a `linkedin_member_id` column to the contacts table to store it:

```sql
ALTER TABLE contacts ADD COLUMN IF NOT EXISTS linkedin_member_id text;
CREATE INDEX IF NOT EXISTS contacts_linkedin_member_id_idx ON contacts (workspace_id, linkedin_member_id);
```

## Import Enrichment

**[VISUAL: Flow diagram — contact import → buildAttendeeMap → scanLinkedIn → match by member ID → patch back → log connection date + message history]**

Webhooks handle live events. Import enrichment handles everything that happened before the integration was connected.

When contacts are imported (CSV, Airtable, manual), we fire a retroactive enrichment job that scans every connected integration for prior history. For LinkedIn, this runs through Unipile's attendee and relations APIs:

`buildAttendeeMap()` fetches all your Unipile chat attendees once and builds four lookup maps: by normalized URL, by member ID, and by connection date indexed against both. We match on member ID first, then URL slug, then fall back to a live Unipile profile fetch that retrieves the `provider_id` and patches it back as `linkedin_member_id` — so the next enrichment run for this contact skips straight to step 1.

We don't do name matching in the enricher. At bulk import scale, name matching produces too many false positives. The webhook handler covers that case: the moment someone messages you on LinkedIn, the name match fires, `linkedin_member_id` gets stored, and all future enrichment passes hit step 1 directly.

## Principles

**The permanent identifier is the only safe match key.** LinkedIn's readable slug is a display alias. Build every identity system around the permanent member ID from day one — it survives URL updates, profile edits, and vanity URL changes.

**Self-healing beats upfront perfection.** Rather than trying to match correctly 100% of the time, patch identifiers back at every step. The system converges to accurate matching over time without any manual intervention. After the first time a contact passes through the waterfall, every future event hits step 1 instantly.

**Non-actionable events must be filtered before any processing.** Any event that contains a recognizable field (like `message_id`) will look actionable unless you explicitly skip it. Enumerate and discard junk events at the top of your handler — before dedup, before resolution, before logging.

**Name matching requires an array query, not a single-row query.** `.maybeSingle()` is correct when you expect at most one result. Once a duplicate exists, it returns `null` — and your create-if-not-found fallback makes things worse. Use an array query and pick the best result yourself.

**Connection date enrichment is as valuable as message history.** Knowing when someone became a connection gives you a signal anchor that most CRMs miss entirely. It's worth storing `connectedAt` per contact and surfacing it in the timeline.
