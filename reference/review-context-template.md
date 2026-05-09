# review-context.md (Template)

Drop a copy of this file at `.github/review-context.md` in any project where you use this specialist regularly. It teaches the reviewer your project's conventions so they stop flagging false positives.

The reviewer reads this **before** reviewing every diff. Keep it tight — only document things that would otherwise be flagged as bugs.

---

# <Project Name> Code Review Context

## Project Overview
<One paragraph: stack, what the app does, where it runs in production.>

## Authentication Model
<Layers of auth, who can access what, which functions are server-only vs public.>

Examples:
- "Dashboard auth: Supabase magic links, JWT validated locally via `getClaims()` (not `getUser()`)."
- "Cron endpoints: `Authorization: Bearer ${CRON_SECRET}`, verified with `crypto.timingSafeEqual`."
- "Public endpoints `/api/subscribe` and `/api/unsubscribe`: no auth, validated by Kickbox / token entropy."

## Database Patterns
<Concurrency strategy, error codes you handle on purpose, RPCs that look weird but are correct.>

Examples:
- "Email uniqueness enforced by Postgres `unique` constraint; we catch error 23505 and return a friendly message. Intentional."
- "`increment_field` RPC is atomic — used for engagement counters under concurrency. Do not flag as a race."
- "Send cron uses `next_send_at = NULL` as a soft lock. `WHERE next_send_at <= NOW()` excludes locked rows."

## Known-Safe Patterns Specific to This Project
<Things that look like bugs but are intentional in this codebase.>

Examples:
- "We use `?? []` defensively because the API occasionally returns `null` for empty arrays. Not over-defensive."
- "`Array.isArray(tags)` checks are intentional — older subscriber rows have `null` instead of `[]`."

## Don't-Flag List
<Specific things that have been re-flagged enough times that we want them silenced.>

Examples:
- "`dynamic import('@supabase/supabase-js')` in route handlers is intentional for cold-start optimization."
- "Service role key in `src/lib/admin/*.ts` files — these are server-side only, never imported into client code."

## Working Style (Reviewer Behavior Rules)
<How the user wants the reviewer to behave. Set during onboarding. The reviewer reads this BEFORE every review and follows it.

  IMPORTANT: capture this section in the user's OWN words, not translated into severity-rubric language. If they said "fix everything because I don't know what I'm doing," write that. If they said "I'm in ship mode," write that. Mirroring their voice is part of how the specialist stays useful instead of becoming jargon.>

Examples (in actual user voice — NOT translated):

For a non-developer:
- "Fix everything you find, even the small stuff. I don't know what's important and I don't want time bombs in my code three months from now."
- "Don't ever decide on your own that something should wait for a later release. Tell me first, I'll decide what waits."
- "Use my words. I say 'lock the splits and get the PDF' — don't change it to 'finalize state transition' or anything fancy."

For a senior developer:
- "Surface everything, I'll triage. Don't filter."
- "Skip the small stuff unless the fix is a clean one-line rename."
- "Ship-mode. Critical and high only."

If the user gave a phrasing that doesn't fit any of these, just paste their phrasing verbatim and let the reviewer read it cold.

## Architecture Boundaries
<Where the trust boundaries are. The reviewer will use this to decide whether validation is missing.>

Examples:
- "All user input is validated at the route handler layer. Server actions assume validated input."
- "Webhook payloads are signature-verified at the edge before reaching handlers."

---

## How to keep this file fresh
- Add a pattern here once it has been verified safe in two independent reviews.
- Remove a pattern here when the code that justified it is removed.
- This file is part of the codebase. PRs that introduce new "looks-like-a-bug-but-is-not" patterns should update this file in the same commit.
