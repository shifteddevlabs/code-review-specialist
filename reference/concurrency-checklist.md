# Concurrency & Lifecycle Checklist

Run this on every diff that introduces or modifies async, concurrent, or lifecycle-triggered code.

## What triggers this checklist
Any of the following appearing in the diff:
- Swift / SwiftUI: `Task {}`, `.task`, `.task(id:)`, `.onChange`, `.onAppear`, `.onDisappear`, `withCheckedContinuation`, `actor`, `@MainActor`, `DispatchQueue.async`, `async let`
- React / TS: `useEffect`, `useLayoutEffect`, `Promise.all`, `await` in a handler, `setInterval`, `setTimeout`, `AbortController`
- Node / server: unawaited promises, fire-and-forget `void someAsync()`, event listeners that mutate shared state
- Any database write that does not use a unique constraint or RPC

## The four questions

### 1. What else fires concurrently from the same trigger?
Search the codebase for sibling handlers of the same lifecycle event.
- Another `.onChange(of: <samevalue>)` in the same view tree?
- Another `.onAppear` on a parent view that also runs this code path?
- Another caller of the same async function that could overlap?

If two handlers can fire simultaneously, document it. If they write to the same state, that is almost certainly a bug.

### 2. What lifecycle / view-state does this assume?
List every assumption the code makes about *when* it runs:
- "View is in the window hierarchy" — guaranteed by `.onAppear`? Not always; SwiftUI can call it on offscreen prefetch.
- "Auth has resolved" — is the auth provider mounted above this view at this point?
- "Sheet has dismissed" — `.onChange` of `isPresented` fires *during* dismissal animation, not after.
- "Component is mounted" — React `useEffect` cleanup runs *after* unmount; setState in a hung promise will warn.

If the code assumes a state that is not actually guaranteed, that is the bug.

### 3. What happens if the call hangs forever?
- Is there a timeout on the network call?
- Does anything else block on this Task / Promise?
- Could a UI overlay (loading spinner, modal) spin forever and trap the user?
- Does the app have a recovery path (retry button, force-quit prompt)?

A hung async call without a timeout is a UX bug waiting for a flaky network.

### 4. What happens if the call fires twice?
- Is the underlying API idempotent? (e.g. `INSERT ON CONFLICT` is, raw `INSERT` is not; iOS HealthKit dedupes by sample identity, but your local cache may not.)
- Is the *side effect* idempotent? (Two analytics events. Two emails sent. Two charges to the user's card.)
- Does the code guard with a flag, a token, or a constraint?

"It only fires once in practice" is not an answer. Concurrency bugs are by definition unlikely until they aren't.

## When to flag
If you cannot give a clean answer to **any one** of the four, flag it. The cost of a false positive on a concurrency finding is one minute of the writer pushing back. The cost of a missed race condition is hours of debugging in production.

## Common patterns and their typical answers

| Pattern | Typical Q1 | Typical Q2 | Typical Q3 | Typical Q4 |
|---|---|---|---|---|
| `.onChange + .onAppear` both load data | Both fire on first appearance + change | Assumes captured ID is still current | No cancellation | Last-write-wins by timing |
| `useEffect` with async fetch, no cleanup | None usually | Component might unmount mid-fetch | setState on unmounted warns | If deps change, fetches stack |
| `Task { await save() }` fire-and-forget | If user clicks Save twice fast, both run | Assumes server is idempotent | Hangs do not block UI but state is wrong | Two saves race on response |
| Webhook handler with no idempotency key | Provider retries on 5xx | Assumes single delivery | Hangs trigger provider retry | Duplicate side effects |

## Resolution patterns
- **SwiftUI:** prefer `.task(id:)` over `.onChange + Task {}`. It auto-cancels.
- **React:** return a cleanup function from `useEffect`; use `AbortController` for fetches.
- **Server:** require an `Idempotency-Key` header on any non-idempotent endpoint that external services can retry.
- **Database:** unique constraints + handling the duplicate error is almost always cleaner than application-level dedup.
