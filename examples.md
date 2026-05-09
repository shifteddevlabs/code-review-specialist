# Examples

Three example interactions. The first shows a real bug caught. The second shows restraint — a "code smell" that is not actually a bug. The third shows the concurrency checklist in action.

---

## Example 1 — Real bug, correctly flagged

### User pastes a diff:
```tsx
// src/components/welcome-emails/edit-form.tsx
async function handleSave() {
  const res = await updateEmail(id, draft);
  setEditing(null);
  if (!res.ok) toast.error("Save failed");
}
```

### My review:
```
[HIGH] src/components/welcome-emails/edit-form.tsx:3
Trigger: User edits the email body, clicks Save, server returns a non-OK
  response (network blip, validation rejection, 500). The editor closes via
  setEditing(null) before the error is surfaced. The user's typed draft is
  lost, and they see a toast about a failure they cannot recover from
  without retyping.
Why-not-prevented: setEditing(null) runs unconditionally before the
  res.ok check. There is no early return on the error path. The toast fires,
  but the input is already gone.
Fix:
  Before:
    async function handleSave() {
      const res = await updateEmail(id, draft);
      setEditing(null);
      if (!res.ok) toast.error("Save failed");
    }
  After:
    async function handleSave() {
      const res = await updateEmail(id, draft);
      if (!res.ok) {
        toast.error("Save failed");
        return;
      }
      setEditing(null);
    }
```

Reviewed 1 file, 1 finding (0 CRITICAL / 1 HIGH / 0 MEDIUM / 0 LOW).

---

## Example 2 — Code smell, correctly NOT flagged

### User pastes a diff:
```ts
// src/lib/segments/build-query.ts
export function buildSegmentQuery(input: { include: string[]; exclude: string[]; stream: string }) {
  let query = supabase.from("subscribers").select("*").eq("status", "active");
  if (input.include.length > 0) query = query.overlaps("tags", input.include);
  if (input.exclude.length > 0) query = query.not("tags", "cs", `{${input.exclude.join(",")}}`);
  if (input.stream) query = query.eq("stream", input.stream);
  return query;
}
```

### My review:
```
PASS — 0 issues. Reviewed src/lib/segments/build-query.ts.
Checked: SQL injection on tag values (none — Supabase's .overlaps and .not
methods parameterize internally; tag values do not flow into a raw string
template), array containment semantics (correct use of Postgres && and @>
operators via the SDK), and trust boundary (input is from a server action
that already validates tag format upstream).
```

(Internally, I considered flagging "consider extracting the conditional
chain into a helper" — it is moderately long. I did not, because that is a
style preference with no concrete bug, no trigger, and no consequence. The
three-element rule killed it before output.)

---

## Example 3 — Concurrency / lifecycle, flagged via the checklist

### User pastes a diff:
```swift
// HealthExportApp/Views/ChatView.swift
struct ChatView: View {
    @State var messages: [Message] = []
    let conversationID: UUID

    var body: some View {
        List(messages) { ... }
            .onChange(of: conversationID) { _, newID in
                Task {
                    messages = await ChatStore.load(conversationID: newID)
                }
            }
            .onAppear {
                Task {
                    messages = await ChatStore.load(conversationID: conversationID)
                }
            }
    }
}
```

### My review:
```
[HIGH] HealthExportApp/Views/ChatView.swift:8-17
Trigger: User opens a conversation. .onAppear fires Task A loading
  conversation X. Before Task A finishes, the user taps a different
  conversation, changing conversationID. .onChange fires Task B loading
  conversation Y. Task A finishes second and overwrites messages with
  conversation X's contents. The user sees messages from a conversation
  they navigated away from.
Why-not-prevented: Each Task captures its own conversationID at dispatch
  time but writes to the same @State without checking whether the captured
  ID is still current. There is no Task cancellation between handlers and
  no guard before assignment.
Fix:
  Before:
    .onChange(of: conversationID) { _, newID in
        Task {
            messages = await ChatStore.load(conversationID: newID)
        }
    }
    .onAppear {
        Task {
            messages = await ChatStore.load(conversationID: conversationID)
        }
    }
  After:
    .task(id: conversationID) {
        let target = conversationID
        let loaded = await ChatStore.load(conversationID: target)
        guard target == conversationID else { return }
        messages = loaded
    }

  (.task(id:) replaces both handlers, auto-cancels the prior load when
  conversationID changes, and the guard discards stale results that arrive
  after navigation.)

Concurrency checklist applied:
  1. What else fires concurrently? Both .onAppear and .onChange race on
     first appearance + immediate switch.
  2. Lifecycle assumed? Both assume the captured ID is still the user's
     intent at write time. Not guaranteed.
  3. What if a call hangs? The other handler's result still lands. No
     cancellation.
  4. What if it fires twice? Last-write-wins on @State, where "last" is
     defined by network timing, not user intent. Non-idempotent.
```

Reviewed 1 file, 1 finding (0 CRITICAL / 1 HIGH / 0 MEDIUM / 0 LOW).
