---
name: apply-changes
description: Apply balancing changes to game data files. Use when a user provides new values, drops files in input/, or describes changes in natural language. Parses input, finds correct TXT files, applies changes, then triggers validation and audit.
---

# Apply Balancing Changes

Parse the user's balancing input and apply it to the correct TXT source files in the game-data repository.

## Trigger

- User provides balancing values (chat, pasted data, or structured request)
- New file detected in `input/` folder
- User says "apply changes", "update values", "change [domain]", or similar

## Process

## Step 0: Invoke alignment-guard (pre-apply mode)

Before parsing input or touching any TXT file, invoke the guard with the change envelope (domain, file, proposed deltas, source of the proposal).

Wait for the verdict:
- **Proceed verdict** → continue to Step 1 (parse input).
- **Revise verdict** → hand the verdict back to the user; do NOT edit.
- **Discuss verdict** → halt; return to design conversation.

Rationale: the guard catches magnitude / shape / scope / assumption drift BEFORE the edit lands. See `.claude/skills/alignment-guard/playbooks/pre-apply.md`.

### 1. Parse Input

Determine what the user wants to change. Supported input formats:
- **Natural language:** "Set Hero A's Skill 1 damage to 450"
- **Pasted tabular data:** Copy-paste from Excel (tab-separated)
- **File in input/:** CSV, Excel export, or text file with values
- **Structured request:** "Here are the new values for [domain]: ..."

Extract:
- Which game system/domain is affected
- Which specific entities are changing (hero IDs, item IDs, building levels, etc.)
- What values are changing (old value if known, new value)

If the input is ambiguous, ask the user to clarify before proceeding.

### 2. Check Repo Freshness

Read the repo paths from the local config. Run:
```
git -C <game-data-path> log --format="%ci" -1
```

If older than 24 hours, warn:
"Your local game-data repo was last updated [time]. Want me to continue with potentially stale data?"

Wait for confirmation before proceeding.

### 3. Locate Target Files

Use `context/file-mappings.md` (index) to identify the domain, then open `context/file-mappings/<domain>.md` for the column schema of the target file(s).
If file-mappings is not yet populated (bootstrapping not done), read the TXT files directly to find the right one.

Confirm with the user:
- "I'll be modifying `[filename]` in the game-data repo. The changes affect [N rows/entries]. Proceed?"

### 4. Apply Changes

#### 4a. Detect file encoding BEFORE choosing an edit method

TXT files in `04_Env_Data/DEV_Data/` are **not all UTF-8**. Roughly 37 of ~209 files are CP949 / EUC-KR (Korean), and one file is UTF-16. The `Read` + `Edit` tool combo silently destroys non-UTF-8 files: `Read` decodes with `errors='replace'` (injecting U+FFFD codepoints into the in-memory string), `Edit` writes back as UTF-8 (encoding U+FFFD as 3-byte `ef bf bd` sequences). Korean text in summary columns becomes garbage; numeric values and IDs survive, so qa-check passes but the file is corrupted.

This is exactly the bug a recent monster-level rebalance pass hit on a CP949 file (thousands of U+FFFD bytes injected across hundreds of rows; caught by accident during commit prep).

**Encoding detector** — run before every edit:

```bash
python3 -c "
import sys
with open('<absolute path>', 'rb') as f:
    b = f.read()
try:
    b.decode('utf-8')
    print('UTF-8 — Edit tool OK')
except UnicodeDecodeError as e:
    print(f'NOT-UTF-8 (fails at byte {e.start}) — use binary splice (4c)')
"
```

#### 4b. UTF-8 file path

If the detector reports `UTF-8`:
- Use the `Read` + `Edit` tool combo as usual.
- Preserve file format exactly (tab separators, column order, line endings).

#### 4c. Non-UTF-8 file path — binary splice recipe

If the detector reports `NOT-UTF-8` (CP949 / EUC-KR / UTF-16):
- **Do NOT use the Edit tool.** It will corrupt the file.
- **Do NOT use the Read tool to inspect the full file** — only use shell tools (`grep`, `awk`, `head`) that operate on raw bytes.
- Use the byte-safe Python binary splice below.

```python
import os
path = "<absolute path to TXT file>"
with open(path, "rb") as f:
    data = f.read()

# For each cell change, define a byte-anchor unique to that row + the new bytes.
# The anchor must include enough surrounding columns (PK + neighbouring cells)
# to be unique in the file. Verify with a count check.
edits = [
    # (old_anchor_bytes, new_bytes)
    (b"\t<row_pk_chain>\t<old_value>\t<next_col>",
     b"\t<row_pk_chain>\t<new_value>\t<next_col>"),
]
for old, new in edits:
    cnt = data.count(old)
    if cnt != 1:
        raise SystemExit(f"anchor not unique: {old!r} appears {cnt} times")
    data = data.replace(old, new, 1)

with open(path, "wb") as f:
    f.write(data)
```

Anchor design tips:
- Lead with a tab character (`\t`) so the anchor lands on a column boundary, not mid-value.
- Include the row's primary-key chain (e.g. `\t<level>\t` for a per-level row) plus the column being changed.
- Append the next column's first few bytes so the anchor doesn't accidentally match a row whose target column happens to share the same value.
- The `cnt != 1` assertion is the safety net — if it fires, lengthen the anchor.

#### 4d. Post-edit verification

After applying the edits (by either method), verify the file's encoding state:

```bash
python3 -c "
import subprocess
path = '<path>'
head = subprocess.check_output(['git','show','HEAD:'+path])
with open(path,'rb') as f: cur = f.read()
fffd = b'\\xef\\xbf\\xbd'
print(f'HEAD U+FFFD: {head.count(fffd)}, current: {cur.count(fffd)}')
print(f'HEAD size: {len(head)}, current: {len(cur)} (delta: {len(cur)-len(head)})')
"
```

Sanity checks:
- **U+FFFD count must not have grown.** If it did, the edit corrupted the file. Revert with `git checkout <path>` and re-do via binary splice.
- **File size delta must be consistent with the intended cell changes.** Non-trivial growth on a CP949 file (e.g. +20% of file size) signals re-encoding to UTF-8. Revert and re-do.

The pre-push hook (`scripts/hooks/check_encoding_regression.py`, wired into `pre_push_validate.sh`) enforces this as a last line of defense — but catching it at edit-time is cheaper and avoids the revert dance.

### 5. Hand off to the matching specialist

Pick a specialist based on the files you just edited. The specialist owns the rest of the pipeline (pre-flight / qa / audit / push validation) — do NOT invoke `qa-check`, `audit-change`, or `scan-code-overrides` yourself when a specialist matches. Single source of truth for the post-edit contract.

| Files touched | Invoke | What the specialist owns |
|---|---|---|
| `Src_Hero_Equip_Data.txt`, `Src_Hero_Completion_Equip_Data.txt`, `Src_Hero_Equip_Pet_Data.txt`, `Src_Equip_*.txt`, `Src_Craft_*.txt`, `Src_Promotion_Hammer_Data.txt` | `equipment-specialist` (Validate mode) | 8-check pre-flight (FK chain, tier / reinforce continuity, override guard, scale-convention, power-column monotonic, pre-push smoke), then qa-check + audit-change |
| Any other `Src_Hero_*.txt` | `hero-combat-specialist-validate` | Pre-flight schema checks (e.g. `_HERO_GRADE.next_tier` ENUM), stat-curve validator, then qa-check + audit-change |
| `Src_Army_Data.txt`, `Src_Require_Data.txt` (troop-cost rows only) | `troop-specialist-validate` | 9-check pre-flight (cost FK chain, hardcoded-ID guard for specific army IDs + family prefix, deprecated-tier re-introduction guard, cost-type guard, class-stat range vs benchmark band, tier monotonicity, author-aid orphan-column INFO, power delegation, pre-push smoke), then qa-check + audit-change |
| Files with a `power` / `cumulated_power` column | `power-specialist-validate` | Per-row `cumulated_power[N] = sum(power[1..N])` invariant for the cumulative tables, magnitude sanity vs benchmark, scale-convention guard, dev-work triggers (formula-living-in-code, two-source-of-truth on equip-tier multipliers), then qa-check + audit-change |
| `Src_World_Monster_Data.txt`, `Src_World_Monster_Level_Data.txt`, `Src_World_Monster_Difficulty.txt`, `Src_World_Monster_Difficulty_Correction.txt`, `Src_World_Monster_Reward_Prob_Data.txt`, `Src_WORLD_MONSTER_BONUS_REWARD_Data.txt`, `Src_World_Group_Set_Monster.txt`, `Src_World_Group_Set_Default.txt`, `Src_World_Group_Layer.txt` | `pve-worldmap-specialist-validate` | 10-check pre-flight (FK chain incl. Group_Layer, broadcast invariant L1-30, level coverage, reward-prob DELTA vs baseline, hardcoded-gate guard with multiple dev-work triggers, difficulty UI coverage, special-event 2× advisory, zero-consumer column flag, pre-push smoke), then qa-check + audit-change |
| `Src_Academy_*.txt` | `research-specialist-validate` | 10-check pre-flight (FK chain incl. category hide-convention, Preset row presence, hardcoded-gate guard for Scout / Tutorial, power monotonicity, encoding), then qa-check + audit-change |
| Anything else | Run `qa-check` → `audit-change` directly (no specialist) | The generic pipeline |

If multiple specialists match (e.g. a hero-equip change that also touches the power column), invoke them in this order: domain specialist (`equipment-specialist` / `hero-combat-specialist` / `troop-specialist`) FIRST, then `power-specialist`, then the generic leaf skills.

If any specialist returns FAIL, STOP. Do not proceed to qa-check. Fix the issue first.

### 6. Generate Change Report

Create a report in `output/reports/` named `YYYY-MM-DD-[domain]-changes.md`:
- Summary of what changed
- Before/after values in a table
- Files modified
- Validation results

(Some specialists produce this as part of their Execute/Validate mode — if so, don't duplicate.)

### 7. Wait for User

Do NOT commit or push. Tell the user:
"Changes applied. [Summary]. Review the changes and let me know when you want to commit."

## Important

- Never modify files in the JSON, client, or server repos — only the game-data repo.
- Only modify TXT files in `04_Env_Data/DEV_Data/`.
- **Data source precedence.** Edits land in `04_Env_Data/DEV_Data/`. If a file is missing there, seed it from the latest patch folder under `00_Data Table/` (read-only fallback) — never write to `00_Data Table/`. If only a legacy file exists in `00_Data Table/`, copy it to `04_Env_Data/DEV_Data/` first, then edit there.
- Always preserve exact file format (tab-separated, encoding, line endings).
- If a change would affect multiple files, list all affected files before proceeding.
- **Encoding-aware editing is mandatory.** ~37 of ~209 DEV_Data TXT files are CP949 / EUC-KR (Korean), and one is UTF-16. Run Step 4a's detector on every target file before editing. Non-UTF-8 files MUST be edited via the Step 4c binary splice — the Edit tool will corrupt them silently. Step 4d's post-edit verifier and `scripts/hooks/check_encoding_regression.py` (pre-push) are defense in depth, not substitutes.
