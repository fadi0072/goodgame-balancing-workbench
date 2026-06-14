# Known Overrides

Catalogue of hardcoded server-side and client-side behaviours that override or constrain data values at runtime. **A TXT change that violates an override here either has no effect or crashes the game.**

Always pre-flight changes for listed entities against this document.

---

## Hero #7 (Brogan) — HP floor

The combat code clamps Hero #7's HP to a minimum of 100 at runtime (`HeroCombat.java:142`).

- Any `hp` value below **100** for `hero_id=7` (at any level) is silently overridden — the change won't take effect.
- At `hp` < **50** for `hero_id=7`, the spawn-protection logic divides by zero and crashes the round. See `INC-2042`.

**Rule:** `Src_Hero_Data.txt` rows where `hero_id=7` must have `hp >= 100`. Validators must halt if this rule is violated.

This is the **only documented override** for this sandbox. Other heroes are not subject to floors.
