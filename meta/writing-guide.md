# Writing Guide — Proply Research

This guide defines the tone, structure, and conventions for everything published in this repo. Use it when writing new docs or reviewing existing ones.

## What This Repo Is

Applied research from building Proply — a CRM memory layer for GTM agents. Every doc here covers something we actually built, a problem we actually hit, and the system we landed on. The goal is that GTM builders, technical founders, and RevOps teams can read these and implement similar systems in their own stack.

Not tutorials. Not postmortems. Documentation of the system as it exists.

## Audience

Someone technical enough to follow code snippets, practical enough to care about implementation over theory, and building something similar. They want to know what the system looks like, why it was built that way, and what they'd need to do to build it themselves.

## Tone

Direct. We write in first person plural ("we built", "we use", "we landed on"). Confident but not boastful. Share the reasoning behind decisions, not just the decisions themselves. Avoid over-explaining things that are obvious to a technical reader.

When something was hard to figure out, say so — but frame it as the nature of the problem, not as a list of our mistakes. The reader wants to understand the challenge so they can avoid it, not read a postmortem.

## Titles

Noun phrases that name the system or concept. Not questions, not "how to" guides, not clickbait.

```
✓  The LinkedIn Resolution Layer
✓  GTM Memory — Contact Signal Capture
✓  Pipeline Stage Computation at the Edge
✗  How We Built a Bulletproof LinkedIn Pipeline
✗  Everything We Got Wrong About Contact Resolution
✗  The Ultimate Guide to LinkedIn Webhooks
```

## Structure

Every doc follows roughly this shape — in this order, no strict headers required for each:

1. **What we're doing** — one short paragraph. The goal, the system, why it exists.
2. **The setup** — tools, architecture, how the pieces connect. One visual here.
3. **The core challenge** — the one or two non-obvious things that made this hard. Not a list of every error — pick the ones that are genuinely interesting or that others will hit too.
4. **The system** — how it actually works. This is the core of the doc. Diagrams, code snippets, the waterfall/flow/schema. One or two visuals here.
5. **How we enriched it** — any second-order logic, retroactive processing, edge case handling.
6. **Principles** — 3–5 takeaways written as observations, not rules. These should be things that only become obvious after building the thing.

## Headers

Use them sparingly. 4–6 per doc maximum. No sub-headers (###). No horizontal dividers (---). If you need a divider, you probably need to restructure.

## Language to Avoid

- "The final solution" → just name the system directly
- "Failure 1, Failure 2" → reframe as the challenge or the constraint
- "We discovered that..." → just say the thing
- "Bulletproof", "seamless", "robust" → meaningless
- Numbered lists for things that aren't inherently sequential

## Code Snippets

Include them when they clarify a concept that prose can't. Keep them short — 5–15 lines. Always explain what the snippet is doing in one sentence above it, not below.

## Visuals

Images are embedded using HTML `<img>` tags, not markdown `![]()` syntax, so width can be controlled:

```html
<img src="../assets/your-image.png" width="60%">
```

Always use `width="60%"`. Never use bare markdown image syntax for assets in this repo — it renders at full width and breaks the visual flow of the doc.


Every doc includes exactly 4 visual placeholders. Mark them clearly:

```
**[VISUAL: short description of what goes here]**
```

Place them at:
1. The architecture/setup overview
2. The core challenge or the "before" state
3. The main system (waterfall, flow, schema)
4. The enrichment or second-order logic

## What Not to Publish

- Specific credentials, workspace IDs, API keys
- Internal team drama or blame
- Speculation about what competitors are doing
- Anything that reads like a sales pitch for Proply

## Prompt for Generating New Docs

When starting a new doc for this repo, use this prompt:

---

*Write a research doc for the Proply Research GitHub repo. The topic is [TOPIC].*

*The doc should cover: what we're building and why, the architecture/setup, the one or two core non-obvious challenges we hit, the system we landed on (this is the core — go deep here), and 3–5 principles that only become clear after building it.*

*Tone: direct, first-person plural, written for a technical GTM builder who wants to implement something similar. Not a tutorial. Not a postmortem. Documentation of the system as it exists.*

*Structure: 4–6 sections, no horizontal dividers, no sub-headers. Title is a noun phrase naming the system.*

*Include exactly 4 visual placeholders marked as `**[VISUAL: description]**` at: (1) architecture overview, (2) core challenge, (3) the main system/waterfall, (4) enrichment or edge case handling.*

*Word count: 800–1400 words.*

*For reference, see the writing guide at meta/writing-guide.md and existing docs in /applied.*

---
