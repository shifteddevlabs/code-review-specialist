# Known Safe Patterns

Patterns that look like bugs but are not. Do not flag these unless the surrounding context changes their meaning.

This is the universal list. Project-specific safe patterns belong in a `review-context.md` at the project root (see `reference/review-context-template.md`).

---

## Database

### `unique` constraint catching duplicates (Postgres error 23505)
Code that does an `INSERT` and catches error code `23505` is using the database as the source of truth for uniqueness — the correct pattern. The race-condition window between `SELECT` and `INSERT` is closed by the constraint itself.

### `.maybeSingle()` instead of `.single()` (Supabase)
Intentional "not found is OK" semantics. Returns `null` instead of throwing. Do not flag as missing error handling.

### `.select()` chained after `.update()` / `.insert()` / `.delete()` (Supabase)
Uses Postgres `RETURNING` clause — single SQL statement, not an extra round-trip. Do not flag as a "redundant query."

### `{ count: "exact" }` in `.select()` (Supabase)
Uses Postgres `COUNT(*)` alongside the data query in one round-trip. Not a separate query.

### `SECURITY DEFINER` RPC returning aggregated data
Safe when the function only returns aggregates (counts, sums, group bys), not raw row data. Flag only if the function returns rows that bypass RLS.

---

## Authentication / Cryptography

### `crypto.timingSafeEqual()`
Timing-safe by design. Do not flag "potential timing attack."

### UUID v4 tokens (122 bits of entropy)
Not guessable. Do not flag "predictable token" on a v4 UUID.

### HMAC signature + timestamp replay protection
Standard webhook verification (Svix, Stripe, GitHub). If both signature and timestamp window are checked, it is correct. Do not flag.

### `SUPABASE_SERVICE_ROLE_KEY` in Route Handlers / API routes
Server-side only — Next.js Route Handlers do not ship server env vars to the client. Flag only if the key appears in client components or `NEXT_PUBLIC_*` vars.

### `getClaims()` (Supabase) for JWT validation
Local validation, no network call. Some projects deliberately use it instead of `getUser()` for performance. Check the project's `review-context.md` before flagging.

---

## Frontend / React

### Null coalescing (`??`) and optional chaining (`?.`)
Intentional null safety, not "missing check." Do not flag.

### `Array.isArray(x)` before array operations
Intentional defense against API responses that occasionally return non-arrays. Do not flag as redundant.

### `useRef` to hold current state for use inside `useCallback`
Avoids stale closures without bloating dependency arrays. Standard pattern.

### Module-level component extraction for static SVG icons
No closure capture, no key/ref issues, no re-render cost. Standard optimization.

### JSX auto-escapes interpolated values
No XSS unless `dangerouslySetInnerHTML` is used. Do not flag interpolated user input as XSS.

---

## Common false-positive triggers

### "Missing error handling"
Before flagging, check whether the function is called from a context that handles errors at a higher level (Next.js error boundaries, server action error wrapping, top-level try/catch in a job runner). Many "unhandled promise" findings are handled — just not in the same function.

### "SQL injection"
If the code uses a query builder (Supabase, Prisma, Drizzle, Knex) or parameterized queries, it is not injectable. Flag only if you see actual string concatenation into raw SQL.

### "Missing rate limiting"
Rate limiting is typically enforced at the edge (Vercel, Cloudflare, an API gateway), not in route handlers. Do not flag a route as "missing rate limiting" without checking whether the project uses edge middleware or a gateway.

### "Hardcoded secrets"
A long string literal is not always a secret. Check whether it is a constant, an enum value, a public identifier, or actually a credential. The Semgrep rule for this is famously noisy.

### "Magic number"
A constant in code is not always a magic number. `200` for HTTP OK, `1000` for ms-to-seconds, `7` for days-in-week — these are universally understood. Flag only if the number's meaning is unclear from context.

---

## How to add to this list
A pattern earns its place here only after it has been independently verified safe in at least one prior review **and** survived at least one production cycle. Same-session additions do not count — that is the writer justifying their own code.
