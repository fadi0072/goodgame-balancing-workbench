# Decisions Log

Append-only history of balancing changes made through this workbench.

Append entries via `audit-change-lite` after every successful commit. Newest entries at the bottom.

---

## 2026-06-14 — Change 1: Mira (hero_id=1) mid-game attack buff at L20

**Commit:** `<this commit>`

**Change:**
- `Src_Hero_Data.txt` · `hero_id=1, level=20` · `attack`: 608 → 700
- `Src_Hero_Data.txt` · `hero_id=1, level=20` · `power`: 1725 → 1909 (recomputed)

**Reason:**
Producer: "Hero #1 (Mira) is feeling a bit weak in the mid-game. Bump her attack
at L20 from 608 to 700. Just one cell." Applied verbatim — request value matched
the current cell (not stale).

**Validation:**
- `qa-check-lite`: PASS — format (601 rows, 13 cols, 0 U+FFFD), range, FK, and
  override-awareness all clean. Power recomputed via `round(hp/10 + attack*2 +
  defense*1.5)` = round(289.6 + 1400 + 219) = 1909, consistent with formula.

**Caveats / follow-ups:**
- Producer said "just one cell," but the workbench rule (apply-changes-lite Step 5)
  requires recomputing the derived `power` column when a stat changes — so two cells
  in the row were touched. Leaving `power` stale would have failed qa-check-lite's
  consistency check. Confirmed with the driver before editing.
- Mira is `hero_id=1`; the only documented override (`known-overrides.md`) is the
  Hero #7 HP floor, which does not apply here.

## 2026-06-14 — Change 2: Tessa (hero_id=11) attack curve +10% at L10–L15

**Commit:** `<this commit>`

**Change:**
- `Src_Hero_Data.txt` · `hero_id=11, level=10–15` · `attack`: ×1.10 each level,
  with `power` recomputed per row. Six rows. Samples:
  - L10 · attack 375 → 413 · power 1062 → 1138
  - L13 · attack 444 → 488 · power 1260 → 1348
  - L15 · attack 490 → 539 · power 1391 → 1489
  (full set: L10 413, L11 438, L12 464, L13 488, L14 515, L15 539)

**Reason:**
Producer: "Hero #11 (Tessa) needs a curve adjustment — boost her attack at L10–L15
by 10% at each level. Six rows in total." Applied attack×1.1 per level; recomputed
`power` per apply-changes-lite Step 5.

**Validation:**
- `qa-check-lite`: PASS — format, range, FK (require_id=5011 untouched, no orphans),
  override-awareness (hero 7 unaffected), and power consistency all clean.

**Caveats / follow-ups:**
- ROUNDING CONVENTION (discovered this change): the producer said "+10%" with no
  rounding rule. L10 (375×1.1 = 412.5) is the only value landing exactly on .5.
  We rounded half-up → 413 (driver-confirmed). NOTE: the dataset's derived `power`
  column is internally consistent under BANKER'S rounding (round-half-to-even =
  Python `round()`, which is what qa-check-lite's `round(...)` formula actually
  means); under that convention 412.5 → 412. Since `attack` is a base stat (not a
  formula-derived column) and its power was correctly derived from 413, this creates
  no integrity violation — but it's a latent gap: `apply-changes-lite` does not pin a
  rounding rule for "+%" boosts, so a future bulk edit could silently disagree with
  the house convention. Flagged for the gap-diagnosis writeup.
- A naive half-up power validator falsely flagged 12 pre-existing .5 rows (e.g.
  hero 1 L1, hero 6 L6) as drift; re-checking with round-half-to-even confirmed the
  baseline data is clean. No baseline rows were modified.

## 2026-06-14 — Change 3: Thorne (hero_id=4) rebucketed unlock_require_id 5010 → 5005

**Commit:** `<this commit>`

**Change:**
- `Src_Hero_Data.txt` · `hero_id=4, level=1–30` · `unlock_require_id`: 5010 → 5005
  (all 30 Thorne rows; no other column touched, power unchanged)

**Reason:**
Producer: "We're rebucketing Hero #4 (Thorne) into the lower-tier unlock chain —
change his unlock_require_id from 5010 to 5005." Applied to all 30 level rows so
the hero's unlock gate is consistent across his progression.

**Validation:**
- `qa-check-lite`: PASS — format, range, override-awareness, power (unchanged) clean.
- FK integrity: PASS. New value 5005 is an existing, populated chain — non-orphan.

**FK chain analysis (the substance of this change):**
- 5010 (left): Thorne was the **sole** member (30 rows, no other hero). The
  file-mappings "update together" rule does NOT trigger — nothing else moves with
  him. After this change **5010 is referenced by zero rows** in the data file.
- 5005 (joined): already shared by Aldric (#2, Epic Warrior), Doran (#10, Common
  Warrior), Roric (#16, Common Warrior). Thorne (Rare Warrior) joins cleanly — all
  Warriors. "Lower-tier" = gating tier, not rarity; no conflict with intent.

**Caveats / follow-ups:**
- ORPHANED REQUIRE-ID: 5010 now has no referencing row. The data file is clean
  (qa-check "can't see the require table"), but the external require table likely
  still defines a now-unused 5010 entry. Hand off to whoever owns the require table
  to retire/repurpose it. Not actionable inside this workbench.
- No cascade edits were needed; Thorne was alone on 5010.
