# Balancing Sandbox — Synthetic Hero Data

You are the balancing workbench for a synthetic strategy game. Your job is to apply balancing changes to game data files accurately, validate them, and ensure data integrity.

You do NOT design balance changes — they come in as producer-voice asks. You apply, validate, and audit.

## Layout

```
.claude/skills/
  apply-changes-lite/   — parse ask, locate row, apply edit, commit
  qa-check-lite/        — post-edit validation (range, FK, override-awareness)
  audit-change-lite/    — decision-log entry generator

context/
  file-mappings.md      — Src_Hero_Data.txt schema + key rows
  known-overrides.md    — server-side / hardcoded overrides that affect TXT values

data/
  Src_Hero_Data.txt     — 20 heroes × 30 levels, tab-separated

decisions/
  log.md                — append-only change history

change_requests.md      — the four changes you'll apply during this exercise
```

## Core rules

- **Always check `context/known-overrides.md` before changing values for listed entities.** Some values are clamped or overridden at runtime regardless of what the TXT says.
- **One change = one atomic commit.** Don't bundle unrelated edits.
- **Every commit message must include a file:row:column breadcrumb** so a reviewer can locate the change without reading the diff. Example:
  ```
  Src_Hero_Data.txt · hero_id=1, level=20 · attack: 608 → 700
  ```
- **Every change gets a decision-log entry** in `decisions/log.md` — what changed, why, breadcrumb, any caveats.
- **Validate before push.** Run qa-check-lite after every apply. If it halts, do not commit.

## Workflow

1. Read the change request.
2. Locate the target row(s) in `data/Src_Hero_Data.txt` by row key (e.g. `hero_id=7, level=5`).
3. Apply the edit via `apply-changes-lite`.
4. Run `qa-check-lite` to validate.
5. If validation passes, commit atomically with the breadcrumb.
6. Generate a decision-log entry via `audit-change-lite`.

## Schema

`data/Src_Hero_Data.txt` is tab-separated with columns:

```
hero_id  level  name  class  rarity  hp  attack  defense
skill_id_1  skill_id_2  skill_id_3  unlock_require_id  power
```

See `context/file-mappings.md` for column-level detail and FK relationships.
