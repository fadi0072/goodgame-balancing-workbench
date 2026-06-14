# File Mappings

## `data/Src_Hero_Data.txt`

Tab-separated. 20 heroes × 30 levels = 600 data rows + 1 header. UTF-8.

### Columns

| # | Column | Type | Notes |
|---|---|---|---|
| 1 | `hero_id` | int | 1–20. Primary key (with `level`). |
| 2 | `level` | int | 1–30. |
| 3 | `name` | string | Hero name; repeated per level row. |
| 4 | `class` | enum | `Warrior` / `Ranger` / `Mage` |
| 5 | `rarity` | enum | `Common` / `Rare` / `Epic` |
| 6 | `hp` | int | Base HP. **Subject to server-side overrides — see `known-overrides.md`.** |
| 7 | `attack` | int | Base attack. |
| 8 | `defense` | int | Base defense. |
| 9 | `skill_id_1` | int | FK to skill table (not provided in this sandbox). |
| 10 | `skill_id_2` | int | FK to skill table. |
| 11 | `skill_id_3` | int or `"null"` | Optional; some heroes have only 2 skills. |
| 12 | `unlock_require_id` | int or `"null"` | FK to require table (not provided). `"null"` = starter hero, no unlock gate. |
| 13 | `power` | int | Derived stat — `round(hp/10 + attack*2 + defense*1.5)`. Update when stats change. |

### Row key

A row is uniquely identified by `(hero_id, level)`. Always cite both in change requests and decision-log entries.

### FK chains

Heroes share `unlock_require_id` values, forming gating chains. When changing an `unlock_require_id`, check whether other heroes reference the same value — they may need updating together.

Currently shared chains (read-only — derive from the data):

- `5005`: shared by multiple Common/Epic Warriors
- `5008`: shared by multiple Common/Rare Mages
- `5011`: shared by two Rangers/Mages
- `5012`: shared by three heroes (one of them is Calla)
- `5018`: shared by two Epic Rangers
- `5020`: shared by two Epic-tier heroes

If you change a row referencing a shared chain, the validator (`qa-check-lite`) will flag orphan / dangling references after the edit.

### Null handling

The string literal `null` (not empty) marks an absent value. When validating, treat `null` as "intentionally absent," not "missing data."

### Encoding

UTF-8. Do not introduce non-ASCII characters into the data columns.
