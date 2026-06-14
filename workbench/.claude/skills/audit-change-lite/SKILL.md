---
name: audit-change-lite
description: Generate a decision-log entry for a change just committed. Use after apply-changes-lite + qa-check-lite pass. Appends to decisions/log.md.
---

# audit-change-lite

Append a structured decision-log entry to `decisions/log.md` summarising a change that has just been committed.

## When to run

After `apply-changes-lite` has committed and `qa-check-lite` has passed. Never before.

## Inputs

- The commit SHA (or the commit's diff if it's the most recent one).
- The change request that motivated the change.
- Any caveats the user wants captured (cascade implications, FK joins, deferred work).

## Output format

Append to `decisions/log.md`:

```markdown
## YYYY-MM-DD — <one-line summary>

**Commit:** `<sha>`

**Change:**
- `Src_Hero_Data.txt` · `hero_id=X, level=Y` · `<column>`: <before> → <after>
  (one bullet per row changed; for bulk changes, summarise the range + cite a sample)

**Reason:**
<1-3 sentences quoting the change request and any context the user added>

**Validation:**
- `qa-check-lite`: PASS (any warnings noted)

**Caveats / follow-ups:**
- (e.g. "Hero #15 also references the same `require_id=5012`; left unchanged per intent")
- (or "None")
```

## Style

- Lead with the breadcrumb so the entry is searchable by file:row:column.
- Capture **why** more than **what** — the diff says what, the log says why.
- If the change had cascade implications that the user navigated, document the decision (e.g. "Hero #14 and #15 also on require_id=5012 — we left them, since they're a separate intentional chain").
- If `qa-check-lite` reported any warnings (non-halting), note them under Validation.

## What this skill does NOT do

- It does not validate. That's `qa-check-lite`'s job.
- It does not commit. The change is already committed by the time this runs.
- It does not push to remote. The user does that.
