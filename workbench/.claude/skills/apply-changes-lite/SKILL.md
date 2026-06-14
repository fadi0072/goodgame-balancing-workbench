---
name: apply-changes-lite
description: Apply a balancing change to data/Src_Hero_Data.txt. Use when a producer-voice change request arrives — parses the ask, locates the target row by row key, applies the edit, and commits atomically. Always pair with qa-check-lite afterwards.
---

# apply-changes-lite

Apply a single balancing change to `data/Src_Hero_Data.txt`.

## Inputs

A change request in plain producer voice, e.g.:

> "Hero #1 (Mira) attack at L20: 608 → 700."

## Steps

1. **Parse the request.** Extract:
   - Row key: `hero_id` and `level` (sometimes a range of levels).
   - Column being changed.
   - Before value and after value (or formula, e.g. "+10%").
2. **Locate the row(s)** in `data/Src_Hero_Data.txt` by `(hero_id, level)`.
3. **Verify the current value** matches the "before" stated in the request. If it doesn't, **STOP and flag** — the request may be stale.
4. **Apply the edit.** Replace only the target cell(s). Preserve tab separators and other columns exactly.
5. **Recompute the `power` column** for any row where `hp`, `attack`, or `defense` changed. Formula: `round(hp/10 + attack*2 + defense*1.5)`.
6. **Show the user the diff** (Before / After per row, with file:row:column breadcrumb) and ask for confirmation before committing.
7. **Commit atomically** with a breadcrumb commit message:
   ```
   Src_Hero_Data.txt · hero_id=X, level=Y · <column>: <before> → <after>
   ```
   One change request = one commit, even if multiple rows were edited.
8. **Hand off to `qa-check-lite`** for validation, and then to `audit-change-lite` for the decision-log entry.

## What this skill does NOT do

- It does not validate. Validation is `qa-check-lite`'s job — invoke it after every apply.
- It does not write decision-log entries. That's `audit-change-lite`.
- It does not push to a remote. The user does that after they've reviewed the commit.

## Output format

After applying, report:

```
Applied: <count> row(s) in Src_Hero_Data.txt
Breadcrumb: <file>:<row-key>:<column>  <before> → <after>
Commit: <commit-sha>
Next: run qa-check-lite, then audit-change-lite.
```
