---
name: qa-check-lite
description: Post-edit validation for data/Src_Hero_Data.txt. Use after apply-changes-lite, or on demand. Halts with explanation on any violation. Reads context/known-overrides.md to catch server-side floor violations.
---

# qa-check-lite

Validate `data/Src_Hero_Data.txt` against format, range, FK, and override rules. Halts on violation.

## When to run

- **After every `apply-changes-lite` invocation.** Always.
- **On demand**, to spot-check the file at any time.

## Checks performed

### 1. Format integrity

- Tab-separated, 13 columns per row.
- 600 data rows + 1 header (no orphaned blank lines).
- UTF-8 encoded; no replacement characters (`U+FFFD`).
- Numeric columns parse as integers (or the literal string `null` where the schema allows).

### 2. Range checks

- `hp > 0`, `attack >= 0`, `defense >= 0` for every row.
- `level` between 1 and 30 inclusive.
- `hero_id` between 1 and 20 inclusive.
- `class` is one of `Warrior`, `Ranger`, `Mage`.
- `rarity` is one of `Common`, `Rare`, `Epic`.
- `power` is consistent with the formula `round(hp/10 + attack*2 + defense*1.5)` per row (flag any drift).

### 3. FK integrity

- For each row with a non-null `unlock_require_id`, verify the value appears in at least one other row OR was introduced intentionally by the current change. New orphan values are flagged but not halted (the validator can't see the require table).
- For each row with a non-null `skill_id_*`, verify the format is an integer in the expected hero-namespaced range (`hero_id * 100 + N`).

### 4. Override-awareness — **load-bearing**

**Read `context/known-overrides.md`. For every documented override, verify the data file does not violate it.**

Concretely, after parsing `known-overrides.md`:

- Find the **Hero #7 (Brogan) HP floor** entry.
- For every row where `hero_id=7`, check `hp >= 100`.
  - `hp < 100` → **HALT with FAIL**. Report which `(hero_id, level)` rows violate.
  - `hp < 50` → **HALT with CRITICAL FAIL** (would crash the server). Report.

This check applies regardless of whether `apply-changes-lite` flagged the change or not — `apply-changes-lite` does not currently load `known-overrides.md`, so this validator is the safety net.

### 5. Regression checks

- Total row count = 601 (1 header + 600 data).
- File size delta vs. previous commit is "reasonable" (within ±10% unless the change brief explicitly expects more).
- No accidental row deletions or duplications.

## Output format

```
qa-check-lite — Src_Hero_Data.txt
  Format:           PASS
  Range:            PASS
  FK integrity:     PASS / 1 new orphan value (require_id=5022 — confirm intentional)
  Override-aware:   FAIL — hero_id=7, level=5 has hp=90, below floor 100 (HeroCombat.java:142)
  Regression:       PASS

Verdict: FAIL — do not commit. Fix the override violation first.
```

## When `qa-check-lite` HALTS

The caller (typically the user driving `apply-changes-lite`) must:

1. **Not commit.**
2. Read the failure reason.
3. Decide: revert the change, OR add the missing rule to `apply-changes-lite` (so future runs catch this pre-edit), OR escalate as a code change.
4. Once resolved, re-run `qa-check-lite` until it passes before committing.

## What this skill does NOT do

- It does not modify files. It reads and reports.
- It does not push to a remote.
- It does not generate decision-log entries (that's `audit-change-lite`).
