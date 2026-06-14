# Phase 1: Skill Critique — apply-changes (Production Version)

## Question 1: In your own words, what does this skill do?

This skill takes a producer's request to change game data values and applies it safely to the TXT files. It reads the desired changes from the producer (e.g. "boost hero attack") and looks for the appropriate line in the data file, then edits it, validates it, and passes it on to specialist validators. There is one important thing though: it doesn't automatically edit. It prevents potentially harmful content from appearing in the document first — such as corrupted encoding on Korean documents and server side changes that would negate the edit. It then forwards to the correct domain specialist (equipment specialist, hero specialist, world map specialist, etc.) to get their own rules thoroughly validated.

## Question 2: What's well-designed about it?

- **The encoding detector (Step 4a) is load-bearing.** It traps an actual nasty bug: CP949 files corrupted silently by UTF-8 tools. Otherwise, Korean text is garbage. The detector is used before the edit and not after - that's the correct defensive approach.

- **Specialist routing (Step 5) distributes validation logic cleanly.** The various validators (equipment, hero, world-map, research) have their own in-depth checks rather than one validator knowing it all. If there is a specialist match, that is the single source of truth for post-edit of a given domain. No confusion as to who is responsible for approving or accepting.


- **The binary splice recipe (Step 4c) is bulletproof for non-UTF-8 files.** It contains anchor-uniqueness checks (assertions for `cnt != 1`), and it contains post-edit verification (count of U+FFFD, file size delta) so that you can detect corruption at edit-time, not at push-time or in production.

- **Step 0 (alignment-guard) gates the entire flow.** Before parsing or touching any file, you invoke a guard with the change envelope. The guard has three clear verdicts: Proceed, Revise or Discuss, which will determine what happens next. This will be smarter than passing on bad changes to validation.

## Question 3: What would you change or add?

- **Note that the binary splice recipe (Step 4c) assumes familiarity with the concept of "byte anchors**  A worked example would reduce mistakes, such as "change a CP949 value on Src_Monster_Data.txt for row X, hero_id=7, level=5" with the actual anchor-bytes displayed. Currently it's accurate but abstract.

- **You'd like a verdict shape to be mentioned in Step 0, but not too much effort is applied to describing them** but not too much effort is applied to describing them. You're outsourcing the decision, which is probably a good thing, but the brief might include some sample verdict shapes. When the producer receives the "Revise", what does the producer see?

- **A huge domain-specific specialist routing table (Step 5).** The table is broken if the specialist goes offline or if another domain is added. A better approach would be to have a registry, or use metadata: each specialist registers itself with the files it knows, and the skill fetches the correct one at runtime.

- **Do not mention rollback/revert in case of qa-check failure. If a specialist returns FAIL** then you stop, but the skill does not say "revert the commit. Should it? The contract is to "do not proceed," with cleanup not specified.

## Question 4: What assumptions does it make about the project around it?

- **If the schema doc for a domain is missing from the file (context/file-mappings/<domain>.md),** then the skill will crash at Step 3, locate target files. It is possible to do bootstrapping without schema but it is not clean.

- **Assumes that there are specialists that are callable by name.** The skill fails silently at Step 5 if there is no `equipment-specialist-validate` or the MCP server is down. There's no graceful degradation to generic qa-check.

- **It assumes that "git" and "git HEAD" will always be "clean** Step 2 executes "git log" to determine freshness. It will fail if the PATH environment variable doesn't contain git or if you're not in a git-tracked directory. The same if there are uncommitted changes.

- **Step 1 checks to ensure that it does match the producer's request**but if the request is stale (someone else has edited the row since it was requested), the skill stops. No automatic recovery, you have to go back to the producer and re-file. Yes, but with a loop added.

- **Makes the assumption that the override guards are in place and up-to-date with the code.** If the code is updated (Hero #7's HP floor goes from 100 to 150), but known-overrides.md remains as 100, then qa-check will not detect it. Manual sync required.
