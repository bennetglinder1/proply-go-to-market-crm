# LinkedIn Contact Resolution Waterfall
### How We Built a Bulletproof LinkedIn Signal Pipeline — and Everything That Broke Along the Way

---

## The Goal

We wanted every LinkedIn touchpoint — messages sent, messages received, new connections — to automatically appear on the right contact in Proply, with the correct timestamp, no duplicates, and no manual work.

The dream: you connect with someone on LinkedIn, send a message, get a reply — and all of it lands in their contact timeline automatically, enriching their pipeline stage and deal health score in real time.

Simple in theory. A complete nightmare in practice.

---

## The Stack

**[VISUAL: Architecture diagram — Unipile → Render backend → Supabase → Proply UI]**

We use **Unipile** as the LinkedIn middleware layer. Unipile connects to your LinkedIn account and pushes webhook events to our backend every time something happens: a message arrives, you send one, a new connection is made. Our Express backend (hosted on Render) receives those webhooks, resolves which contact they belong to, and writes them to `contact_activity_log` in Supabase.

From there, the Proply UI surfaces them in the contact timeline and uses them to drive pipeline stage progression automatically.

---

## What We Were Starting From

Before this work, LinkedIn signals were largely invisible. You could import contacts with a LinkedIn URL, but there was no live connection between what was happening in LinkedIn and what appeared in Proply. No messages, no connection dates, no signal that someone had replied or gone cold.

The integration existed in skeleton form — Unipile was connected, a webhook route existed — but almost nothing was actually working end-to-end.

---

## The Failures (In Order of Discovery)

### Failure 1 — Webhooks hitting the wrong server

The first thing we discovered: Unipile was configured to POST webhooks to `goproply.com` — our **frontend** domain. The frontend has no webhook handler. Every LinkedIn event was silently dropped before it ever reached our backend.

**Fix:** Updated Unipile's webhook URL to point directly to the Render backend:
```
https://assetly-backend-pwz2.onrender.com/webhooks/linkedin?workspace_id=<id>&secret=<secret>
```

---

### Failure 2 — Every webhook returning 401

Once we fixed the domain, every webhook returned `{"error":"invalid_secret"}`. The `WEBHOOK_SECRET` environment variable existed in our local `.env` but had never been added to Render's environment dashboard.

**Fix:** Added `WEBHOOK_SECRET` to Render. First working webhook log appeared within minutes.

---

### Failure 3 — Wrong field names

With webhooks now arriving, contact resolution still failed. We were reading `sender.profile_url` and `sender.name` — but Unipile's actual payload uses different names:

```
sender.attendee_profile_url   ← NOT sender.profile_url
sender.attendee_name          ← NOT sender.name
sender.attendee_provider_id   ← the permanent LinkedIn member ID
```

**Fix:** Updated all field references to match Unipile's actual payload shape.

---

### Failure 4 — `message_read` events poisoning the dedup system

Unipile fires non-actionable events alongside real ones: `message_read`, `message_delivered`, `message_edited`, `message_reaction`. These payloads contain a `message_id` field — which our `isMessage` detection was reading as a real incoming message.

The result: a `message_read` event would trigger the dedup check, find the `li_msg_<id>` already logged, and write a "Dedup skip" — causing the actual message that followed to also be skipped.

**Fix:** Added a `NON_ACTIONABLE` set that short-circuits the handler before any contact resolution:
```js
const NON_ACTIONABLE = new Set([
  'message_read', 'message_delivered',
  'message_edit', 'message_edited',
  'message_delete', 'message_deleted',
  'message_reaction'
]);
if (NON_ACTIONABLE.has(eventType)) return res.json({ ok: true, skipped: eventType });
```

---

### Failure 5 — Backfill never ran for outbound messages

When a new contact is created from a LinkedIn webhook, we immediately backfill their full message history. But outbound message webhooks don't include the recipient's LinkedIn URL — only the sender's. So `linkedinUrl` was null, the backfill guard `if (created && linkedinUrl)` blocked it, and history was silently lost.

**Fix:** Unipile always provides `body.chat_id` on every event. We now pass that directly to the backfill function, bypassing the need for a URL entirely:
```js
if (created) {
  const chatId = body.chat_id || body.provider_chat_id || null;
  backfillLinkedInMessages(supabase, workspaceId, contact.id, { linkedinUrl, chatId });
}
```

---

### Failure 6 — Duplicate contacts snowballing

We kept ending up with 2–3 records for the same person. Root cause: contacts are often imported from CSV/airtable without a `linkedin_url`. When a LinkedIn webhook arrives for that person, the URL match fails (no URL stored), and the name match used `.maybeSingle()` — which returns `null` when more than one row matches. Step 3 (create) then fires and creates a new duplicate.

Once you have 2 contacts with the same name, every future webhook creates a 3rd. Snowball.

**[VISUAL: Diagram showing the duplicate snowball — 1 contact → import → 2 contacts → webhook → .maybeSingle() → null → 3 contacts]**

**Fix:** Replaced `.maybeSingle()` with an array query. When multiple name matches exist, prefer the record with a `linkedin_url` already set (more data = more likely the canonical record), then fall back to oldest `created_at`:
```js
const { data: nameMatches } = await supabase
  .from('contacts')
  .select('*')
  .eq('workspace_id', workspaceId)
  .ilike('first_name', first)
  .ilike('last_name', last)
  .order('created_at', { ascending: true });

const byName = nameMatches?.find(c => c.linkedin_url) || nameMatches?.[0] || null;
```

---

### Failure 7 — The LinkedIn URL format mismatch

This was the deepest problem. LinkedIn has **two completely different URL formats** for the same person:

| Format | Example | Source |
|---|---|---|
| Readable slug | `/in/georgi-furnadzhiev` | Imported from CSV, airtable, manual entry |
| Member ID | `/in/ACoAADL-V-kBZ5Dd-k90zDpKft0ia3Qr36MFn1M` | What Unipile sends in message event payloads |

These strings have **nothing in common**. `ilike '%/in/georgi-furnadzhiev%'` will never match a stored URL of `/in/ACoAADL...`. So a contact imported from CSV with a readable slug would fail URL match every single time a webhook arrived for them — and fall through to create a new duplicate.

**[VISUAL: Side-by-side showing the two URL formats and how they don't match]**

**Fix:** Added a `linkedin_member_id` column to the contacts table. The member ID (`ACoAA...`) is LinkedIn's permanent internal identifier — it never changes, regardless of whether the person updates their vanity URL. Unipile always provides it as `sender.attendee_provider_id` on message events and `user_public_identifier` on connection events.

```sql
ALTER TABLE contacts ADD COLUMN IF NOT EXISTS linkedin_member_id text;
```

This became the primary match key. Any time we touch a contact via LinkedIn, we write this ID back. All future lookups hit it instantly.

---

### Failure 8 — Supabase query builder has no `.catch()`

After wiring up the member ID patching logic, webhooks started throwing:

```
TypeError: supabase.from(...).update(...).eq(...).catch is not a function
```

Supabase's `PostgrestFilterBuilder` implements `.then()` (making it `await`-able) but does **not** implement `.catch()` as a standalone method. Our patch calls used `.catch(() => {})` chained directly on the query, which throws synchronously before `await` even runs.

**Fix:** Replace `.catch(() => {})` with `.then(null, () => {})` on all Supabase update queries where errors should be swallowed:
```js
// ❌ Broken
await supabase.from('contacts').update(patch).eq('id', id).catch(() => {});

// ✅ Fixed
await supabase.from('contacts').update(patch).eq('id', id).then(null, () => {});
```

---

## The Final Solution: The Resolution Waterfall

After all of the above, we landed on a 4-step contact resolution waterfall that runs on every LinkedIn webhook event. Each step self-heals by patching the identifiers it discovers back onto the contact — so the system gets faster and more accurate over time.

**[VISUAL: Waterfall diagram with 4 steps, each with a fallback arrow]**

```
resolveLinkedInContact(supabase, workspaceId, { linkedinUrl, fullName, memberId })
```

### Step 1 — LinkedIn Member ID (permanent)
```js
contacts.linkedin_member_id = memberId  // ACoAA...
```
Fastest. Always works once the ID has been seen once. Survives URL format changes, profile URL updates, anything. If found, patches `linkedin_url` back if missing.

### Step 2 — LinkedIn URL slug match
```js
ilike contacts.linkedin_url '%/in/<slug>%'
```
Catches contacts imported with a readable slug URL. Patches `linkedin_member_id` back so step 1 works next time.

### Step 3 — First + last name match
```js
ilike first_name + ilike last_name → prefer record with linkedin_url
```
Safety net for contacts with no LinkedIn data at all. Uses array query (not `.maybeSingle()`) to handle existing duplicates gracefully. Patches both `linkedin_url` and `linkedin_member_id` back.

### Step 4 — Create
Creates a new contact as a last resort. Immediately patches `linkedin_member_id` onto the new record.

---

**The self-healing property is key.** After the first time any contact passes through this waterfall, their `linkedin_member_id` is stored. Every future event for them hits step 1 instantly — no URL matching, no name matching, no API calls. The system gets faster the more it's used.

---

## Import Enrichment — Retroactive History

**[VISUAL: Flow diagram — import → enrichContactHistory fires → Gmail + LinkedIn + Calendar scan]**

When contacts are imported (CSV, airtable, or manual), we immediately fire a retroactive enrichment job that scans every connected integration for prior history. For LinkedIn specifically:

1. **`buildAttendeeMap()`** — fetches all your Unipile chat attendees once, builds two lookup maps:
   - `byUrl` — normalized LinkedIn URL → Unipile attendeeId
   - `byMemberId` — LinkedIn member ID → Unipile attendeeId

2. **`scanLinkedIn()`** — for each imported contact, tries in order:
   - Match by `linkedin_member_id` (step 1)
   - Match by URL slug (step 2)
   - Live Unipile profile fetch to get `provider_id` (Path B) — and patches it back as `linkedin_member_id`

The enricher is the import-time equivalent of the webhook waterfall. It doesn't have a name match step (too fuzzy at bulk scale), but the webhook handler covers that case: the moment someone messages you on LinkedIn, the name match fires, `linkedin_member_id` gets stored, and all future enrichment runs hit step 1.

---

## What This Unlocks

With the full pipeline working:

- **New connection** on LinkedIn → appears in contact timeline with the real connection date
- **Message sent or received** → logged instantly, pipeline stage advances to Interested or Engaged
- **New contact from LinkedIn** → full message history backfilled in the background
- **Imported contacts** → past Gmail threads, LinkedIn messages, and calendar meetings pulled retroactively
- **Dedup script** → one command cleans up any duplicates that accumulated before these fixes: `node scripts/dedup-contacts.mjs "Proply WS"`

---

## Key Lessons

**1. Middleware field names are never what you expect.**
Assume the payload shape is wrong until you've logged it and confirmed every field name against a real event. Unipile's docs and actual payloads diverged on multiple fields.

**2. The URL format problem is fundamental, not a bug.**
LinkedIn's two URL formats aren't a Unipile quirk — it's how LinkedIn's own infrastructure works. The readable slug is a vanity alias. The member ID is the real identifier. Build around the permanent ID from day one.

**3. `.maybeSingle()` is a silent trap at scale.**
It's designed for queries that should return at most one row. The moment you have a duplicate in the DB (and you will), it returns `null` instead of the best match — and your "create if not found" fallback creates another duplicate. Use array queries and pick the best result yourself.

**4. Self-healing beats perfect upfront matching.**
Rather than trying to get the match right 100% of the time on the first attempt, patch identifiers back at every step. The system converges to perfect matching over time without any manual intervention.

**5. Non-actionable webhook events will ruin your dedup.**
Any event that contains an ID field will look like an actionable event unless you explicitly skip it. Always enumerate and skip the junk events before doing any real processing.
