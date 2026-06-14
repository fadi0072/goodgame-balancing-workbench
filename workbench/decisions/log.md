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
