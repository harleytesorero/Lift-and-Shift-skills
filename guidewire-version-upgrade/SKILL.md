---
name: guidewire-version-upgrade
description: Structured workflow for upgrading a Guidewire PolicyCenter LOB (Line of Business) product from one platform version to another (e.g., Niseko to Quasar). Uses the three-way delta extraction approach — extracting customer-only changes from the source version and applying them to the clean target OOTB. Includes LOB removal, migration phases (WS-0 through WS-7), compile/runtime error resolution, and hardening. Only trigger when the user explicitly asks to "upgrade Guidewire," "migrate a LOB to a new version," "run the NtoQ upgrade," "tech upgrade," or "version upgrade" — do NOT trigger for general Guidewire development or bug fixing.
---

# Guidewire Version Upgrade Skill

A phased workflow for migrating a Guidewire PolicyCenter LOB product from an older platform version (source) to a newer platform version (target). Based on proven execution of Niseko-to-Quasar IM migration.

## Explicit invocation only

This skill does not self-trigger. Only proceed when the user explicitly asks for a version upgrade, tech upgrade, or LOB migration between Guidewire platform versions.

## Core Principle

**The target OOTB is the authoritative base. You add customer customizations to it — never the other way around.**

> "Do not upgrade the entire source repository in place and do not overwrite target OOTB with source. Treat target OOTB as the authoritative new base."

### The Delta Formula

**(Source + LOB) minus (Source OOTB) = LOB customer changes only**

Then apply those customer changes to the clean Target OOTB.

### Three Required Codebases

| Ref | What | Example |
|---|---|---|
| Source OOTB | Clean baseline of the old version | `C:\IS_APPS\PC-N` (Niseko OOTB) |
| Source + LOB | Old version with LOB customizations | `C:\IS_APPS\AAIS\PC` (Niseko + IM) |
| Target OOTB | Clean baseline of the new version | `C:\IS_APPS\gw-releases\PC` (Quasar OOTB) |

## The Golden Rule: OOTB + LOB, OOTB Wins

For every file that exists in both Target OOTB and Source+LOB:

```
RESULT = Target OOTB (base, untouched) + Customer LOB additions (appended/inserted)
```

- **Start from the Target OOTB version.** Never start from the Source+LOB version.
- **Add** customer LOB entries (typecodes, entity fields, display keys, LocationRefs, Gosu methods) **to** the OOTB version.
- **If a duplicate exists** (same key, same entry in both OOTB and LOB), **keep the OOTB version, discard the LOB duplicate.**
- **Never replace** an OOTB file with a LOB file. Never `cp` a Source+LOB file over a Target OOTB file.
- **Never bulk-restore** (`git checkout HEAD -- modules/configuration/`) to fix individual issues — this wipes all legitimate merges. Restore individual files only.
- **Always ask the user** before modifying any OOTB file. Explain what LOB content is being added and why.

This rule applies to ALL file types: PCFs, Gosu, typelists, entities, properties, plugin configs, integration schemas, rules, systables — everything.

## File Classification Rules

Before touching ANY file, classify it:

| Classification | Action |
|---|---|
| Customer-created (`cust/*`, `*_Ext.*`, LOB entities, LOB PCFs, LOB product model) | Copy to target, adapt to target APIs |
| Customer extension of OOTB (modified `.properties`, shared PCFs, shared Gosu with LOB additions) | Three-way merge: apply LOB intent to the target version |
| Modified OOTB file (OOTB Gosu/config changed for LOB) | Do NOT copy. Reapply the business intent to target's version |
| Copied Guidewire core code | Do NOT transplant. Reimplement using target's extension points |
| Generated artifact (CSS, JS, class files, deploy/) | Regenerate from target toolchain |
| Unrelated LOB/customization (non-LOB client code, other LOB remnants) | Exclude |
| Shared client utility (SurePath, integrations) | Migrate ONLY if LOB has a proven dependency |
| Third-party library/JAR | Confirm technical compatibility, licensing/reuse rights, security/CVE status, and target support before adding |

## Critical Rules

1. **Do NOT overwrite target OOTB** — no full-tree copy or force replacement
2. **Do NOT transplant Guidewire core code** — port customer behavior only
3. **Do NOT use Source+LOB vs Target as the only diff** — that mixes customer changes with platform changes
4. **When target and source disagree, target wins** — re-express business requirement using target-supported extension points unless the official Update/Release guidance says otherwise
5. **Every migrated file must be traceable to a LOB requirement or dependency**
6. **OOTB > LOB — ALWAYS.** When a file exists in both Target OOTB and Source+LOB, the result must be **OOTB base + LOB additions**. Start from the Target OOTB version, then add only the customer LOB entries to it. NEVER replace the OOTB version with the LOB version. If there are duplicate entries (same key, same typecode, same LocationRef, etc.), keep the OOTB version and discard the LOB duplicate.
7. **Do NOT hide release-specific conflicts.** Record and resolve them against applicable Guidewire Update/Release guidance. Do not assume a successful compile means the upgrade is correct — release-specific behavior/API changes can require a semantic rewrite even when the code compiles.
8. **Do NOT import client secrets or data.** Configuration secrets, credentials, sample customer data, environment-specific values, and real client/customer PII are not migration artifacts. Use synthetic/non-sensitive test data only.
9. **Keep modernization separate.** Do not convert the LOB implementation to APD merely because the target version supports APD, and do not adopt GO products, unless that is an explicit business objective. APD/GO modernization is a separate decision from the technical upgrade.
10. **Do NOT preserve obsolete customizations.** If the target now provides equivalent OOTB behavior, decide whether the customization is still required. Do not carry forward dead customizations by default.
11. **Third-party libraries must be vetted.** Confirm technical compatibility, licensing/reuse rights, security/CVE status, and target platform support constraints before adding any third-party library or JAR.
12. **NEVER use bulk directory copy (`cp -r source/. target/`) for ANY directory that contains OOTB files.** This blindly overwrites Target OOTB files with Source+LOB versions. For EVERY directory migration (PCFs, rules, plugins, integration, config, etc.), you MUST: (a) list the customer delta files, (b) classify each as customer-created vs. customer-modified-OOTB, (c) copy only customer-created files, (d) three-way merge customer-modified OOTB files. Bulk copy is ONLY safe for directories that are entirely customer-created (e.g. `gsrc/cust/`, `gsrc/ext/`, LOB product model directories).
13. **If you must modify a Target OOTB file, ALWAYS ask the user first.** The only valid reasons to modify an OOTB file are: adding LOB-specific entries (typecodes, entity fields, LocationRefs, display keys), or restoring it after an accidental overwrite. Always tell the user which OOTB file is being modified and why.
14. **Only migrate files that are LOB or LOB-related.** Every file migrated must meet one of these criteria: (a) LOB-specific — IM entities, IM Gosu, IM PCFs, IM product model, (b) LOB-related — shared code that IM directly depends on, (c) Referenced by LOB — files that IM code calls/imports, (d) LOB references them — files that reference IM types. Do NOT migrate general customer customizations (DBA, SurePath, CLUE, CONVR, MVR, etc.) unless IM has a **proven dependency**. Before migrating any non-IM file, ask: "Does IM code directly reference this?" If you can't prove the dependency, skip it.

## Migration Guard (MUST CHECK BEFORE EVERY ACTION)

Before EVERY file copy, edit, or fix:
1. Ask: "Is this file LOB-related?" If NO, do NOT touch it
2. Ask: "Will this fix introduce a NEW dependency chain?" If YES, stop and evaluate
3. Ask: "Have I seen this error pattern before?" If YES, apply the known fix
4. **ALWAYS tell the user the plan before executing**

Before COPYING any file to Target OOTB:
5. Ask: "Does this file already exist in Target OOTB?" If YES, do NOT copy — you must MERGE instead (start from Target OOTB version, add LOB entries)
6. Ask: "Is this file in a directory that contains OOTB files?" If YES, do NOT bulk-copy the directory — classify and process each file individually
7. If the file does NOT exist in Target OOTB, it is customer-created — safe to copy directly

Before MODIFYING any file in Target OOTB:
8. Ask: "Is this file from Target OOTB (exists in git HEAD)?" If YES, you MUST ask the user for permission before modifying. Explain what LOB entries you are adding and why.
9. The result must always be: OOTB content preserved + LOB additions appended. NEVER remove or change existing OOTB content.
10. If there are duplicate entries between OOTB and LOB, keep OOTB and discard LOB.

Before REMOVING a file to fix a runtime error:
11. Ask: "Is it in Target OOTB?" If YES, do NOT remove
12. Ask: "Is it in Source OOTB?" If YES, do NOT remove
13. Ask: "Was it added during the LOB migration?" If YES and not LOB-related, remove it
14. If it is NOT in Source+LOB either, it might be from Target OOTB, do NOT remove

Before copying ANY file: check if it is in the LOB-specific directories or directly called by LOB code. If not, skip.
Before copying ANY typelist: check if LOB entities/product model reference it. If not, skip.
Before fixing ANY compile error: check if the erroring file is LOB-related. If not, ignore it.

## Token Efficiency

These rules minimize token consumption without sacrificing correctness. The highest-cost
patterns in upgrade workflows: per-file agent analysis of the delta inventory, re-reading
files to verify edits, spawning agents for obvious compile fixes, and running verification
after every phase.

- **Batch delta extraction with diff tools, not agents.** Use `diff -ruN` to generate the
  full customization inventory in one pass. Don't spawn agents to read and compare files
  individually. Parse the diff output to classify files mechanically (added/modified/deleted).
- **Minimize agent spawning for compile error resolution.** For obvious fixes (missing import
  → add it, `override` keyword mismatch → fix it, removed property → comment out), use direct
  Edit tool calls. Only spawn agents for errors requiring multi-file reasoning or business
  logic judgment.
- **One verification pass per gate, not per fix.** After resolving a batch of compile errors,
  run `gwb compile` once. Don't recompile after each individual fix. Group fixes by phase,
  verify at the gate.
- **Classify files mechanically before reading them.** The diff output tells you whether a
  file is customer-created (not in Source OOTB) or customer-modified (in both). Only read
  customer-modified files that need three-way merge. Customer-created files can be copied
  without reading.
- **Bulk copy customer-created directories in one command.** `gsrc/cust/`, `gsrc/ext/`, LOB
  product model directories — use `cp -r` (or `xcopy`) directly. Don't read or analyze each
  file.
- **Don't re-read files to verify copies or edits.** The file tools confirm success. Only
  re-read when doing a three-way merge (need to see both Target OOTB and Source+LOB content)
  or when validating XML well-formedness after block insertion.
- **Compact agent prompts.** Don't repeat Critical Rules, Migration Guard, or File
  Classification Rules in agent task prompts — the agent system prompt already has them.
  Prompts should contain: phase, file list, action per file (copy/merge/adapt), exit criteria.
- **Parallelize independent phases.** WS-1 (typelists) and WS-2 (product model) can run in
  parallel if they touch different files. WS-5 (PCF/UI) and WS-6 (integrations) can overlap
  similarly. Max 3 concurrent agents.
- **Cache the OOTB file list.** Run `git ls-files` on Target OOTB once at Phase 0 and cache
  the result. Use it for all "does this file exist in OOTB?" checks instead of per-file
  lookups.
- **Known error patterns: fix directly, don't research.** The Known Error Patterns table
  already has the fix for each pattern. When a compile error matches a known pattern, apply
  the fix directly — don't spawn an agent to investigate.

## Effort Level Guidance

Claude cannot change its own effort level — **tell the user** when to switch (`/effort`).
Default to **medium**; ask for **high** only where reasoning quality changes the outcome.

| Phase | Effort | Why |
|---|---|---|
| **Phase 0 planning, delta strategy, OOTB baseline** | **High** | Errors here propagate through every WS phase |
| Delta extraction, file classification from diff output | Medium | Mechanical diff tooling |
| WS phases: bulk copy of customer files, config merges with clear diffs | Medium | Mechanical |
| **Three-way merges where customer and OOTB both changed the same file** | **High** | Requires understanding intent of both changes |
| Compile errors matching the Known Error Patterns table | Medium | Fix is already documented |
| **Compile/runtime errors not in the table, or failing a second fix attempt** | **High** | Non-obvious root cause |
| **Server startup failures (product model, bizrules, REST mapping, DB upgrade)** | **High** | Error detail is often buried above the stack trace |
| LOB removal within the upgrade | Follow `pc-plan-lob-removal` guidance | — |
| Hardening, final verification, reports | Medium | Mechanical |

Prompt the user at each transition, e.g.: "Phase 0 complete — you can switch back to
`/effort medium` for WS-1 bulk copy."

## Dependency Scope Rule

When copying shared dependencies, copy ONLY the specific files LOB code directly references — not entire directories. Every migrated file must be traceable to a LOB requirement.

Examples of over-broad copies to AVOID:
- Entire `cust/lob/common` directory when only one file is needed
- All ScriptParameters when only 2 are LOB-specific
- Entire SurePath framework when only one helper is used
- **`cp -r` of `config/web/pcf/`, `config/rules/`, `config/plugin/`, `config/integration/` or ANY directory containing OOTB files** — this overwrites hundreds of Target OOTB files with old Source+LOB versions, causing cascading orphan references, missing API calls, and wrong platform behavior. This mistake was made during Niseko-to-Quasar IM migration and caused 388 OOTB PCFs to be overwritten.

## Safe Directory Copy Rule

**Directories that are SAFE to bulk-copy** (entirely customer-created, no OOTB files):
- `gsrc/cust/` — customer Gosu source
- `gsrc/ext/` — customer extensions
- LOB product model directories (e.g. `InlandMarineLine/`)
- `content/cust-ratebooks/` — customer ratebooks
- `config/extensions/displayname/` customer displayname files (if not in OOTB)

**Directories that MUST be processed file-by-file** (contain OOTB files):
- `config/web/pcf/` — PCF files (mix of OOTB and customer)
- `config/rules/` — rule files (mix of OOTB and customer)
- `config/plugin/` — plugin registrations (OOTB)
- `config/integration/` — mappings, schemas, updaters (OOTB)
- `config/locale/` — display.properties and variants (OOTB)
- `config/resources/` — systables, product config (OOTB)
- `gsrc/gw/`, `gsrc/edge/`, `gsrc/web/`, `gsrc/wsi/` — platform Gosu (OOTB)
- Any other directory where Target OOTB has existing files

For every file-by-file directory: list customer delta → classify each file → copy customer-created → merge customer-modified-OOTB (OOTB base + LOB additions, OOTB wins duplicates).

## Pre-Start: Ask the User for the Three Codebase Paths

The paths in "Three Required Codebases" are **examples only — never assume them.** Before any
work, ask the user (one AskUserQuestion call or one message) for:

1. **Source OOTB** path + version/patch (e.g. Niseko 2025.07 P5)
2. **Source + LOB** path + which LOB(s) are being migrated
3. **Target OOTB** path + version/patch
4. **Output/upgrade branch** location (new folder or branch off Target OOTB)

Then verify each path exists (`ls <path>/gwb.bat <path>/modules/configuration`) before Phase 0.
If any path is missing or the version can't be identified, stop and ask — do not substitute.

## Build Commands — Default: AI Runs Them, Filtered

**Default: run `gwb` yourself (in the background, filtered).** It is the cheapest option and
the user is fine with the wait. Only ask "who runs builds" if the user has not stated a
preference. If the user runs them, ask them to paste ONLY the filtered error lines (see below),
never the full log.

See **Running gwb Builds (Token-Efficient)** below for exactly how.

## Running gwb Builds (Token-Efficient)

Applies to `gwb clean`, `gwb codegen`, `gwb compile` and `gwb runServer`.

- **Where logs go:** a migration-logs folder **outside** the target repo. Never write a
  MIGRATION folder into the repo. Example:
  `C:\IS_APPS\Updated-PC\Upgrade Logs\<Project>-MIGRATION\`. Use this layout from the start:

  | Path | Contents |
  |---|---|
  | `MIGRATION_MANIFEST.md`, `PHASE0_TRACKER.md` | Root: the authoritative records |
  | `builds/codegen/codegenN.log` | One file per codegen run, numbered |
  | `builds/compile/compileN.log` | One file per compile run, numbered |
  | `builds/server/serverN.log` | One file per runServer attempt, numbered |
  | `migrated-files/` | `wsN_copy.txt`, `compileN_copy.txt` and removal lists (files copied or deleted per step) |
  | `phase0/` | Delta inventory and classification lists (delta.tsv, created_*, merge_*, scan_*) |
  | `PHASE0_REPORT.md` | Root: a **script-generated** summary of phase0/ (counts, meaning, file per step) |

  Token rule: write lists with scripts (they cost about zero tokens) and read back only counts.
  Generate reports by script from those lists. Never hand-write large reports or dump list contents.

  Never overwrite an earlier run's log. Bump N instead, so each manifest "Compile run N" section
  matches `compileN.log`.
- **How (Windows):** run it from PowerShell with `run_in_background`, calling the batch file
  directly. `cmd /c "gwb ..."` failed with "not recognized" in this harness. Example:
  `Set-Location <target>; [Environment]::CurrentDirectory=(Get-Location).Path; & .\gwb.bat compile *> "<logs>\builds\compile\compileN.log"; "EXIT $LASTEXITCODE"`
  **Don't run redundant builds.** `gwb compile` already runs the codegen tasks (gen*Sources), and
  `gwb runServer` already runs codegen and compile (verified in the CR7 logs). Never run
  `gwb codegen` before compile. During the runtime-fix loop, run only `gwb runServer` and grep its
  log for compile errors as well as startup errors. Run `gwb compile` on its own only for a
  compile-only gate (G2–G4) when no server start is planned.
  Then read only the error lines:
  `grep -naE "error:|FAILED|BUILD (SUCC|FAIL)|Caused by|What went wrong" <log> | head -150`
- **Time:** on a full CR7-sized codebase, codegen takes 4–10 minutes and compile 20–30 minutes.
  If compile is killed for low memory, check for stray Gradle/Java processes before rerunning.
  Don't poll and don't sleep; you are notified when the background run finishes. The user doesn't
  need to do anything meanwhile.
- **Why it's cheaper:** an error batch costs a few dozen lines instead of hundreds of lines of
  Gradle stack trace. A pasted full log (~25k+ tokens) stays in context and is re-sent on every
  later turn. Never read the full log, and never ask the user to paste it.
- **Dedupe before fixing:** collapse repeated errors to one line per distinct message/type,
  e.g. `grep "error:" log | sed -E 's/.*error: //' | sort | uniq -c | sort -rn`. 84 copies of
  one missing type are one fix.
- **Error threshold:** gosuc aborts at 100 errors. Fix by root cause, recompile, repeat.
  Before recompiling, scan for every used-but-undefined `_Ext` symbol in one pass, so each
  compile batch is smaller.
- **runServer (one limit):** it is long-running and needs watching. Start it in the background
  writing to a log, then use Monitor (or a background `until grep -qE "<ready line>|ERROR|Exception" log`)
  so you are notified on the ready line or on startup failures. Read only the matched lines
  plus ~30 lines of context, not the whole log. It takes a little more back-and-forth than compile.
- **Always run in the correct repo root** (the Target / upgrade branch). Verify with
  `ls gwb.bat modules/configuration` first. A build run in the wrong folder gives false gate results.

## Pre-Work Assumptions — Confirm Before Starting

Before any migration work begins, confirm these assumptions with the user:

| ID | Assumption |
|---|---|
| A1 | The team has a legally/contractually authorized right to reuse the source client's LOB code and associated artifacts. |
| A2 | The exact source release/patch level of the customized code can be identified. |
| A3 | A matching Source OOTB/base can be obtained, or a reliable original baseline tag/commit exists in source control. **Do not substitute a different patch casually.** A mismatched old base will create false positives and false negatives in the customization inventory. |
| A4 | The Target OOTB source is pristine and its release/patch identity is known. |
| A5 | The team can access the applicable Guidewire PolicyCenter Update/Release guidance for Source→Target conflicts that are not fully covered by public docs. |
| A6 | The primary objective is functional preservation of the existing LOB line, not redesign. |

If any assumption cannot be confirmed, raise it with the user before proceeding. Do NOT start migration work without A1–A4 confirmed.

## Execution Phases

### Phase 0: Preparation (WS-0)
1. Freeze and verify all three inputs (Source OOTB, Source+LOB, Target OOTB)
2. **Do NOT run or wait for a Target OOTB baseline build.** The Target OOTB is a pristine Guidewire release and is assumed to compile. Instead, verify pristineness with `git status` (0 uncommitted changes) and that no customer code exists (`gsrc/cust`, LOB folders absent). If a later gate shows an error in a file the migration never touched, check it against the Target's git HEAD before treating it as a migration issue.
3. Generate the LOB customization inventory using `diff -ruN` between Source OOTB and Source+LOB
4. Classify every item in the inventory using the file classification rules
5. Build the **migration tracker** with these fields for every artifact:

| Field | Purpose |
|---|---|
| Path / artifact | Exact file, object, rule, entity, product artifact, or integration component |
| Change type | Added / modified / deleted / generated / dependency |
| Domain | Data model / product / Gosu / rating / rules / UI / forms / integration / API / config / test |
| Source baseline counterpart | Whether the artifact exists in Source OOTB |
| Target counterpart | Equivalent Target artifact or 'none' |
| Recommended action | Copy / reapply / merge / rewrite / retire / exclude |
| Dependency | Other custom artifact(s) required by LOB |
| Owner | Developer/SME responsible for resolution |
| Test | How the migrated behavior will be verified |
| Status / decision | Not started / in progress / done / blocked + decision note |

6. **Capture golden test scenarios** — before migration, record a small set of representative LOB scenarios from the Source with known inputs/outputs: selected clauses, modifiers, key rule outcomes, premium/cost outputs, forms/UW outcomes, and integration requests. Use synthetic/non-sensitive data only.
7. Mark items as LOB / required shared dependency / unrelated / excluded. Present to user for scope review.
8. Create the upgrade branch from Target OOTB

### Phase 1: Migrate Typelists and Data Model (WS-1)
1. Copy customer-created LOB typelists (.tti and .ttx)
2. Copy customer-created LOB entities (.eti and .etx)
3. **MERGE** modified OOTB typelists/entities — add only LOB-specific entries to the Target version, do NOT copy the Source version
4. Add missing typelists referenced by LOB entities
5. **Review every custom foreign key, array, delegate, index, validation, and lifecycle assumption** for Target compatibility — do not assume Source structural patterns are valid in Target
6. Regenerate or refresh generated artifacts using the Target toolchain where applicable
7. **Gate:** `gwb codegen` must succeed — compile immediately after the structural model is stable; do not wait for UI/rating work

### Phase 2: Migrate Product Model (WS-2)
1. Copy entire LOB product model directory
2. Copy LOB ratebooks, systable data files, display names
2a. **Copy LOB lookup table registration file** from `config/lookuptables/` (e.g., `lookuptables.{LOB}.xml`). This file maps lookup table names to their entity subtypes. **Without it, the product model `-lookups.xml` files will fail with 856+ "No lookup table found" errors at runtime.** This was missed in the IM migration and caused a server crash.
3. Map all product sub-artifacts: coverables, exposures, coverages, exclusions, conditions, modifiers, clauses, availability/existence behavior, and edition/jurisdiction logic as applicable
4. **MERGE** shared product config — add only LOB-specific entries to Target versions. Validate dependencies on shared product constructs or package/line containers; bring them only when required.
5. **APD decision:** Do NOT convert the LOB implementation to APD merely because the Target supports APD or offers a GO product for this line. Unless the current implementation is already APD-based or the business explicitly chooses redesign, keep APD/GO adoption as a separate modernization decision.
6. **Gate:** `gwb codegen` must succeed

### Phase 3: Migrate Gosu/Domain Code (WS-3)
1. **SAFE COPY** LOB-specific Gosu (`gsrc/cust/`, `gsrc/ext/`) — these directories are entirely customer-created, bulk copy is safe
2. **FILE-BY-FILE** for `gsrc/gw/`, `gsrc/edge/`, `gsrc/web/`, `gsrc/wsi/`:
   a. List customer delta (files that differ between Source OOTB and Source+LOB)
   b. For each file: check if it exists in Target OOTB
   c. If NOT in Target OOTB → copy from Source+LOB (customer-created)
   d. If IN Target OOTB → three-way merge: start from Target OOTB version, add LOB-specific changes. **NEVER copy Source+LOB version over Target OOTB.** OOTB wins on conflicts.
3. Copy shared dependencies ONLY if LOB needs them (check each one)
4. Adapt to Target APIs — compile errors expected, classify each as:
   - Missing customer dependency → copy the specific file
   - Changed GW API → adapt the code
   - Removed/deprecated API → find the Target replacement
   - Changed package visibility → use Target's extension points
5. **Add/update unit/GUnit coverage** around business-critical custom behavior while resolving incompatibilities — do not defer all testing to WS-7
6. **Fix WS-3 compile errors in this phase. Defer errors that belong to later phases:**
   - Display key errors → defer to WS-5 (locale file merges)
   - Non-LOB references in shared files (e.g. CLUE, MVR, NAICS in shared cust/ code) → defer to WS-7 (trim/stub)
   - Fix errors that ARE WS-3 scope: missing customer Gosu deps, platform API changes, override mismatches, removed platform types
7. **Common WS-3 fixes (from Niseko-to-Quasar experience):**
   - **`ScriptParametersEnhancement.gsx`**: Copy the customer enhancement, then **trim to LOB-only properties**. Non-LOB ScriptParameter properties (IVANS, SmartComm, etc.) will fail because the params aren't registered. Remove them.
   - **Platform JAR types removed in Target**: Some Source Gosu files extend or import platform classes that no longer exist in Target JARs (e.g. `gw.rating.basic.lob.im.*`, `gw.lob.common.dependency.*`). **Remove** the file if no LOB code depends on it, or **stub** it if LOB code does depend on it (return empty/default values).
   - **Override methods that no longer exist in Target base class**: The Target base class may have removed extension methods (e.g. `createActivities_Ext`, `runPreQuote_Ext`, `executeOnStart_Ext`). Change `override function` to `function` (regular method).
   - **Property removed from Target entity**: If a property no longer exists on a Target entity (e.g. `.Description` on a schedule cov item), comment out the reference.
   - **Missing `gw/lob/common` customer packages**: LOB code may import customer-added packages in `gw/lob/common` (e.g. `annotations`, `statcode`, `util`). Copy only the specific packages LOB needs — do NOT copy the entire `gw/lob/common` customer tree. Check transitive deps of each package.
   - **Missing enhancements**: If LOB code uses a property or method not on the base type (e.g. `valueSafelyAsDigits_Ext`), search for the `.gsx` enhancement file that provides it and copy it.
   - **Missing shared PCF helper classes**: LOB PCFs may reference customer utility classes (e.g. `CovTermTypkeyInputSetHelper`, `APDRulesHelper_Ext`). Copy the specific file and verify its imports.
8. **Gate:** `gwb compile` must succeed (run `gwb clean` first). Remaining display key errors and non-LOB shared file errors are acceptable at this gate if they are tracked for WS-5/WS-7.

### Phase 4: Migrate Rating and UW (WS-4)
1. **FILE-BY-FILE** for rules, systables, question sets, lookup tables:
   a. Customer-created files → copy directly
   b. Files that exist in Target OOTB → MERGE (OOTB base + LOB additions)
2. Copy LOB rating code, cost data, modifiers, custom rating service adapters (customer-created only)
3. Migrate underwriting issue/referral rules, validation, availability, and eligibility logic
4. **Validate against golden scenarios** captured in Phase 0 — compare Target behavior (inputs/outputs) against the pre-migration baseline. Where Target's base behavior changed, **document whether the difference is an intended Target behavior change or a regression** in the migrated custom logic.
5. **Gate:** `gwb compile` must succeed; agreed rating/UW scenarios pass

### Phase 5: Migrate UI/PCF (WS-5)
1. **FILE-BY-FILE** for `config/web/pcf/`:
   a. Customer-created PCFs (not in Target OOTB) → copy directly
   b. OOTB PCFs modified by customer → MERGE: start from Target OOTB version, add LOB-specific widgets/logic. **NEVER copy Source+LOB PCF over Target OOTB PCF.**
   c. If an OOTB PCF needs a LOB LocationRef, ask user before modifying
2. **MERGE** LOB display keys into Target's display.properties — append customer-only keys to the OOTB file. De-duplicate: if key exists in OOTB, keep OOTB version.
3. Confirm removed/renamed widget/API usage — check that PCFs don't reference Source-era widgets or APIs that Target has changed or removed
4. **Validate end-to-end LOB wizard/navigation flow only AFTER backend/model compilation is stable** (Phases 1-4 gates passed). Do not attempt UI validation before domain code compiles.
5. **Gate:** `gwb compile` must succeed

### Phase 6: Migrate Forms and Integrations (WS-6)
1. **FILE-BY-FILE** for `config/integration/`, `config/plugin/`:
   a. Customer-created files → copy directly
   b. OOTB files → MERGE (OOTB base + LOB additions) or leave as OOTB
2. Copy only integrations LOB actually calls
3. Review plugin interfaces and custom integration contracts for Target API changes
4. **Review custom REST endpoints and Cloud API extensions** against current Guidewire Cloud Standards. GW security guidance discourages custom servlets — point integrations toward supported REST mechanisms.
5. Review messaging destinations, event payloads, document/forms inference, external rating/doc services, and any custom auth assumptions
6. Copy LOB forms inference config (customer-created only)
7. **Use mocks/contract tests** where external systems are unavailable — do not block source readiness on full environment integration when a reliable contract test can prove compatibility
8. **Gate:** `gwb compile` must succeed; required external contracts compile and pass contract tests/mocks

### Phase 7: Quality and Hardening (WS-7)
1. Fix remaining compile errors deferred from earlier phases
2. Start server and test LOB functionality end-to-end:
   - Server starts clean (MULTIUSER run level)
   - LOB product appears in product selection
   - Submissions can be created
   - Quoting works with correct rate books
   - Coverage UI renders correctly (required fields, visibility, auto-fill)
   - Binding/issuance works (payment plans, billing plugin)
3. Fix runtime errors as they appear
4. **Common WS-7 runtime issues (check ALL of these on every migration — they are always missed):**
   - **Required field markers missing**: The shared OOTB PCFs `CovTermInputSet.default.pcf` (and bit, datetime, decimal, longtext, Option, Package, shorttext, Typekey variants) and `CovTermDirectInputSet.pcf` use `required="term.Pattern.Required"`. The Source+LOB changes this to `required="cust.pcf.coverage.CovTermRequirednessInputSetHelper_Ext.isRequired(term)"`. Copy the `CovTermRequirednessInputSetHelper_Ext.gs` helper and merge the `required=` attribute into each Target PCF.
   - **Coverage fields not auto-filling**: The shared OOTB `CoverageInputSet.default.pcf` PostOnChange uses OOTB `APDRulesHelper`. The Source+LOB changes `disablePostOnEnter` to use `cust.pcf.coverage.APDRulesHelper_Ext.hasDependents`. Merge this change into the Target version.
   - **Additional coverage fields showing too many fields**: `CovTermDirectInputSetHelper.gs` is an OOTB file modified to add `IM_NON_VISIBLE_COVTERMS_EXT` and `NON_EDITABLE_COVTERMS_EXT` lists. Copy the Source+LOB version if Target OOTB = Source OOTB for this file.
   - **Manuscript endorsement NPE**: `IMManuscriptEndorsement_ExtPanelSet.pcf` passes null coverable to `CoverageInputSet` when no endorsement is selected. Add `visible="manuscriptEndorsement.IMManuscriptEndorsementCov != null"` to the `InputSetRef`.
   - **Missing lookuptables config**: The `config/lookuptables/lookuptables.{LOB}.xml` file must be copied. Without it, product model import fails with hundreds of "No lookup table found" errors.
   - **Missing question set dependencies**: IM question sets may reference shared entity extensions (e.g., `IndustryGroup_Ext` on `Account`). These must be migrated even though they're not IM-named, because IM's question set availability scripts use them.
5. **Run Cloud Assurance / Studio inspections.** Review migrated code for:
   - Deprecated APIs → replace with Target-supported equivalents
   - Hidden/internal packages → remove reliance on packages not intended for customer customization
   - Direct database access → remove unsupported direct SQL patterns, use supported InsuranceSuite mechanisms
   - Integration framework → align custom integration approaches with current supported REST/integration mechanisms
   - OSS/SCA → review customer-introduced dependencies for vulnerabilities and supportability
   - Guidewire Layer → ensure migration did not modify/copy core Guidewire-layer artifacts
   Treat security/compliance validation as part of the migration, not a final cleanup activity.
5. **Inventory customer-introduced third-party libraries/JARs.** Confirm they remain needed, check licensing/reuse rights, run Software Composition Analysis where available, and remediate known vulnerabilities.
6. Document all findings

## Release-Specific Review

Source and Target are not necessarily adjacent release baselines. The team does not need to create intermediate applications, but it MUST account for applicable product/configuration changes introduced between the two endpoints.

| Review step | Direction |
|---|---|
| Source → intermediate releases | Review applicable Update/Release guidance for changes touching artifacts in the LOB migration inventory |
| Intermediate → Target | Continue the same impact review through the Target version |
| Collision handling | When a Target artifact/API differs from Source, use the release guidance to choose the supported Target pattern |
| No-collision items | Do NOT perform speculative refactoring merely because a release changed elsewhere |
| Evidence | Link each non-trivial remediation to the relevant Guidewire update/release item or a documented technical decision |

**What NOT to do:** Do not assume that successfully applying a text patch means the upgrade is correct. Release-specific behavior/API changes can require a semantic rewrite even when the code can be made to compile.

When reviewing, use this worksheet structure per impacted item:

| Release change | Impacted LOB artifact | Guidewire action | Our action | Owner | Test |
|---|---|---|---|---|---|
| `<Update item>` | `<path/object>` | `<required/recommended change>` | `<merge/refactor/none>` | `<role>` | `<test case>` |

## Validation Gates

Never migrate every file first and then attempt one giant compile. Incremental gates reduce the search space: structural errors are resolved before Gosu errors, domain errors before UI errors, and code issues before functional acceptance.

| Gate | Focus | Exit Condition |
|---|---|---|
| G0 | Pristine Target baseline | Target OOTB verified pristine via `git status` (no baseline build required) |
| G1 | Structural model | Typelists + data model + essential product model validate/compile |
| G2 | Domain | Core LOB Gosu/domain code compiles; missing dependencies are explicit |
| G3 | Business behavior | Rating, rules, UW, validation behave correctly for agreed golden scenarios |
| G4 | Experience & interfaces | PCF/UI, forms, required integrations/APIs compile and pass targeted tests |
| G5 | Cloud compliance | Cloud Assurance/Studio inspections reviewed; blocking migration issues resolved or formally accepted |
| G6 | Functional acceptance | Required LOB transaction scenarios complete; known differences documented |
| G7 | Handover | Migration tracker, tests, decision log, and source branch are complete and reviewable |

Map phases to gates: Phase 0 → G0, Phases 1-2 → G1, Phase 3 → G2, Phase 4 → G3, Phases 5-6 → G4, Phase 7 → G5–G7.

## Golden Test Scenarios and Behavioral Equivalence

### Capture BEFORE migration (during Phase 0)
Record a small number of representative Source LOB scenarios with known:
- Inputs and selected clauses/modifiers
- Key rule outcomes
- Premium/cost outputs
- Forms/UW outcomes
- Integration requests (where practical)

Use synthetic or non-sensitive data. Do NOT move real client/customer PII into the Target dev workspace.

### Minimum functional smoke suite (during Phase 7)

| Scenario | Minimum check |
|---|---|
| Account / policy context | Create or select account; start the applicable LOB submission/product |
| Risk | Add the core LOB risk/coverable and required attributes |
| Coverage | Select required/optional coverages; exercise important availability conditions |
| Validation | Trigger key validation rules and expected error/warning paths |
| Rating | Quote representative risks and compare agreed premium/cost outputs against golden scenarios |
| Underwriting | Exercise at least one material referral/UW issue path if the implementation uses it |
| Bind/issue | Complete bind/issue where required by the target demonstration/acceptance scope |

### Transaction regression
Add only the policy transactions required by the LOB scope: policy change, renewal, cancellation/reinstatement, rewrite. Do not broaden the acceptance suite without a business requirement.

### Handling Q behavior differences
Where the Target legitimately changes base behavior, record the accepted expected difference rather than forcing exact binary equivalence.

## Definition of 'Target-Ready'

| Exit Area | Criterion |
|---|---|
| Baseline integrity | Target Guidewire base remains authoritative; no Source Guidewire-layer code has been transplanted |
| Scope completeness | All required LOB customer artifacts and necessary shared dependencies are present |
| Compile | Target application/configuration compiles with the migrated LOB changes |
| Product/data validation | Typelists, entities/extensions, product model, and generated/derived artifacts are internally consistent |
| Cloud compliance | Cloud Assurance / Studio inspections have no unapproved blocking violations attributable to migrated code |
| Functional smoke | Required LOB scenarios execute through the agreed business checkpoints |
| Regression | Critical LOB behavior, rating outputs, validation/UW behavior, and key integrations match agreed expectations |
| Traceability | Every migrated artifact is mapped to its Source customization and has an owner/test disposition |
| Known differences | Any intentional Target behavior change or deferred remediation is documented and accepted |

## Git Execution Model and Commit Discipline

Use a source-control structure that makes the technical merge auditable and reversible.

### Recommended branch/tag conventions

| Branch/tag | Purpose |
|---|---|
| `ref/source-ootb` | Read-only reference for the matching Source OOTB base |
| `ref/source-lob` | Read-only reference for the customized Source LOB state |
| `ref/target-ootb` | Read-only reference for pristine Target OOTB |
| `upgrade/lob-source-to-target` | Working branch created from Target OOTB receiving the migrated customer changes |

### Commit rules
- **Commit in functional slices** (typelists/model, product model, domain code, rating, UI) — not a single mass commit
- **Reference migration-tracker IDs** in commit messages
- **Do not combine modernization/refactoring** with compatibility changes unless required for Target support
- **Keep rejected/obsolete Source customizations** documented in the decision log rather than silently omitting them
- **Preserve original Source references** — do not rewrite the source evidence used for the merge

### Pull-request review questions
Before merging any PR in the upgrade branch, answer:
1. Is every change attributable to a LOB requirement or required dependency?
2. Does this PR modify or transplant any Guidewire core layer code?
3. For modified OOTB artifacts, was the Target version used as the base?
4. Is there a release/update item that changes the correct Target implementation?
5. Are new shared dependencies justified and minimal?
6. Are tests attached for the migrated behavior?
7. Are secrets, client data, or environment-specific values excluded?

## Known Error Patterns

### Compile Errors
| Pattern | Cause | Fix |
|---|---|---|
| `StructuredLogger.XXX` not found | Source had custom logger subcategories not in Target | Replace with `StructuredLogger.CONFIG` or `LoggerCategory` |
| `No property descriptor found for property X` | Entity extension field from non-LOB code | Remove the non-LOB entity extension |
| `ClassNotFoundException` | Missing implementation class referenced by entity | Check if LOB-related; if not, remove the entity extension |
| `No function defined for X_Ext` | Enhancement method from Source not in Target | Check if LOB needs it; if yes, port the enhancement |
| `cannot be converted to Type[]` | ArrayList to array mismatch in Target API | Add `.toTypedArray()` |
| `override` missing | Target compiler stricter than Source | Add `override` keyword |

### Runtime/Startup Errors
| Pattern | Cause | Fix |
|---|---|---|
| `ProductModelImportException` | Systable data referencing missing entries | Fix systable references or restore OOTB version |
| Rate table `multiplicand is null` | Rate book missing data for UW company | Add correct UW companies to systables |
| `NullPointerException` in billing plugin | OOTB `StandAloneBillingSystemPlugin` WSDL enum bug | Replace `.GosuValue` calls with string literals |
| Missing display key | LOB code references display key not added to properties | Add the display key to display.properties |
| `ScriptParameters` not found | Stale codegen cache | Run `gwb clean` then `gwb codegen` |
| Coverage fields not auto-filling | PCF PostOnChange wired to OOTB helper instead of custom | Wire to custom `APDRulesHelper_Ext` |
| Required field markers missing | PCF required attribute using OOTB logic | Wire to custom `CovTermRequirednessInputSetHelper_Ext` |
| Rate routine not found | Wrong rate book version active or missing rate book | Import correct rate books, set status to Active |

### Runtime/Startup Errors — OOTB Overwrite Pattern
| Pattern | Cause | Fix |
|---|---|---|
| `Could not find plugin implementation type X` | OOTB plugin .gwp file overwritten by Source+LOB version pointing to old class name | Restore .gwp from Target OOTB git, re-copy customer-only .gwp files |
| `Missing implementations for expected extension interfaces` | OOTB extensioninterfaces.properties overwritten — missing new Quasar interface mappings | Restore from Target OOTB git, merge customer entries back |
| `Page must have a parent or be included in a LocationGroup` | OOTB LocationGroup PCF (e.g. Desktop.pcf) overwritten — lost new Quasar LocationRefs | Restore from Target OOTB git, add customer LocationRefs back |
| Multiple missing plugin/interface errors on startup | Entire `config/plugin/` or `config/extensioninterfaces/` was bulk-copied from Source+LOB | Restore ALL OOTB files in the directory from git, re-add customer-only files |

### CRITICAL: Never Use `git checkout HEAD --` on the Whole Module
Running `git checkout HEAD -- modules/configuration/` restores ALL tracked (OOTB) files to their pristine state, which **wipes out all legitimate customer merges** (entity merges, typelist merges, display key additions, Gosu three-way merges). This happened during the Niseko-to-Quasar IM migration and required re-doing all WS-1 through WS-6 merge work. If you need to restore specific OOTB files, restore them **individually** — never the whole module.

### H2 Database Rules
- Delete H2 (`pc.mv.db`) whenever entity schema changes (`.eti`, `.etx`, `.tti`, `.ttx`)
- Do NOT delete H2 for systable data changes — those auto-sync on startup
- Rate books stored in H2 — must re-import after H2 delete
- Runtime properties stored in H2 — must reconfigure after H2 delete

## Migration Manifest (REQUIRED)

At the end of each workstream (WS-1 through WS-7), **append to a manifest file** called `MIGRATION_MANIFEST.md` at the root of the migration-logs folder, which sits outside the repo (see the "Where logs go" layout above). This is the authoritative record of every file touched and is required for replicating the migration to another project.

### Why This Is Mandatory

When a migration needs to be replicated to a different project (e.g., from a dev workspace to a bitbucket-connected project), relying on filename patterns (`IM*`) or memory will miss files. Examples of files missed during the IM migration replication:
- `InlandMarineStatRecord_Ext.eti` — doesn't start with `IM`
- `IMCoverageNotes_ExtPanelSet.pcf` — under `pcf/surepath/` not `pcf/line/im/`
- Shared OOTB files needing merge (locale files, `PolicyPeriod.etx`, `Account.etx`, `UWCompanyCode.ttx`, `ScriptParameters.xml`, `underwriting_companies.xml`) — must be merged, not wholesale copied

### Manifest Format

For each workstream, record every file in one of three categories:

```markdown
## WS-X: <Phase Name>

### Copied (customer-created, safe to copy as-is)
- `config/extensions/entity/IMBaileeCustomers.eti`
- `config/extensions/entity/InlandMarineStatRecord_Ext.eti`
- `gsrc/cust/lob/im/rating/CustIMRatingEngine_Ext.gs`
- ...

### Merged (OOTB file with LOB entries added — must merge, NOT copy)
- `config/extensions/entity/PolicyPeriod.etx` — added: IMTransactions array, IM_RATING_WS_Ext event
- `config/extensions/typelist/UWCompanyCode.ttx` — added: uwc_IM1/IM2/IM3 typecodes + PVUWCompCode_Ext typefilter
- `config/locale/display.properties` — appended: 220 IM display keys
- ...

### Modified (OOTB file with config/behavior change — must re-apply change, NOT copy)
- `config/plugin/registry/GeocodePlugin.gwp` — disabled="true", applicationKey=""
- `config/config.xml` — UseGeocodingInPrimaryApp="false"
- `gsrc/gw/plugin/billing/impl/StandAloneBillingSystemPlugin.gs` — replaced InvoiceItemType.Installment.GosuValue with InstallmentKey
- ...
```

### When Replicating to Another Project

Use the manifest as a checklist:
1. **Copied files** → copy directly from the completed migration project
2. **Merged files** → restore the target project's OOTB version from git, then apply only the listed additions
3. **Modified files** → restore the target project's OOTB version from git, then re-apply only the listed changes

**NEVER wholesale-copy OOTB files** (locale, config.xml, config.properties, entity extensions like PolicyPeriod.etx) from one project to another — the OOTB base may differ between projects/versions.

## LOB Removal Protocol

If the source build has multiple LOBs and you need to isolate one, follow this protocol exactly. **No shortcuts.** Past LOB removals (CP7, CA7, WCM, GL7, CR7, CU, BP7, CPP) produced 50+ user-caught errors because the agent skipped steps. Every step below exists because a real failure proved it necessary.

### Iron Rules — Violations WILL Cause Compile/Runtime Failures

1. **Consumer-first deletion.** NEVER delete a definition (constant, typecode, entity, display key, typelist, enhancement) without first grepping the ENTIRE codebase for all consumers. Remove every consumer, THEN delete the definition, THEN grep again to verify zero references remain. This rule was violated on EVERY LOB removal — the same file (`UnderwriterEvaluatorTriggeringPoint_Ext.gs`) broke 3 times across 3 different LOBs from this exact mistake.
2. **Start-boundary grep only.** Use `\bWCM` NOT `\bWCM\b`. The end boundary misses compound identifiers like `WCMLocation`, `WCMLargeDeductible`, `WCMonopolistic`. Filter false positives manually after.
3. **Match key names, not entire lines.** When grepping display/properties files, match on the key name (before the `=`) only — auto-generated hash IDs can contain LOB abbreviations by coincidence. A false positive on a hash containing `bp7` deleted a valid Restaurant QuestionChoice and crashed server startup.
4. **Sequential agent execution only.** NEVER run definition-deleting agents in parallel with reference-updating agents. Agent A (update/remove references) completes fully before Agent B (delete definitions) starts. Parallel execution orphaned references on every LOB removal.
5. **Never modify platform-owned files.** Platform files include: `gw/webservice/*` API Javadoc, `config/metadata/*`, base `.eti`/`.tti`/`.tix` files, `display_*.properties` translations (except main `display.properties` for customer keys), `typelist_*.properties` translations. If a grep hits only platform files, warn the user and skip. 45 files were wrongly modified during past removals.
6. **Trace subsystems, not individual files.** Individual file deletion missed 20+ files in WCM alone. For each LOB, trace the full dependency chain: entities → batch processes → event rules → messaging destinations → plugins → PCFs → schedulers → display names → systable registrations → webservice endpoints → Gosu templates → mock data → test fixtures.

### Phase R-0: Discovery — Build the LOB Inventory

Before deleting anything, discover EVERYTHING the LOB owns:

1. **Find all LOB entities and typelists.** Grep for the LOB prefix across `.eti`, `.etx`, `.tti`, `.ttx`, `.tix` files.
2. **Find all LOB Gosu code.** Search `gsrc/` recursively for files containing LOB-specific types, constants, or imports.
3. **Find all LOB PCF/UI files.** Search `config/web/pcf/` for files named with the LOB prefix or containing LOB-specific references.
4. **Find all LOB product model files.** Search the product model directory for the LOB.
5. **Find all LOB configuration registrations.** Check EVERY one of these locations:

**Mandatory Search Directories — EVERY LOB removal must sweep ALL of these:**

| Category | Paths |
|---|---|
| Entities & Typelists | `config/extensions/entity/`, `config/extensions/typelist/` — `.eti`, `.etx`, `.tti`, `.ttx`, `.tix` |
| Product Model | `config/resources/productmodel/`, `config/resources/ratebooks/`, `config/resources/systabledata/` |
| Systable Registrations | `config/resources/systables.xml`, `config/resources/systables.CUST.xml`, **any** `systables.*.xml` variant |
| Rules & Events | `config/rules/**/` — `.gr` files AND their parent `order.txt` entries |
| Plugin Registry | `config/plugin/registry/**/*.gwp` |
| Messaging | `config/messaging/messaging-config.xml` |
| Scheduler | `config/scheduler/scheduler-config.xml` |
| Display Properties | `config/locale/display.properties`, `config/locale/display_en_US.properties`, `config/locale/display_yy.properties` |
| Schema Display | `config/locale/schema.display.properties`, `config/locale/schema.display_en_US_EDG.properties`, `config/locale/schema.display_en_US_EDG_PH.properties` |
| Typelist Properties | `config/locale/typelist.properties`, `config/locale/typelist_en_US.properties`, `config/locale/typelist_yy.properties`, `config/locale/typelist_es.properties`, `config/locale/typelist_es_ES.properties` — **all locale variants** |
| Product Model Display | `config/locale/productmodel.display.properties` |
| Smoketests | `config/smoketests/typelists.properties`, `config/smoketests/**` |
| State Sets | `config/statesets/StateSets.xml`, `config/statesets/**` |
| Resource Context | `config/resources/ResourceContextMap.xml` |
| Webservice | `config/integration/`, `config/web/`, `*.wsdl`, `*.wsc`, `*.xsd` |
| REST API Schemas | `integration/**/*.schema.json`, `config/integration/**` |
| APD Config | `config/apd/`, `config/pca_config/` |
| OSGI | `config/osgi/` |
| Import | `config/import/` |
| Content | `config/content/` |
| Display Name Files | `config/extensions/displayname/` — `.en`, `.es`, and all locale variants |
| Gosu Source | `gsrc/**/*.gs`, `gsrc/**/*.gsx`, `gsrc/**/*.gst` (templates — runtime crash risk) |
| Gosu Tests | `gtest/**/*.gs`, `gtest/**/*.gsx` |
| Mock Data | `**/mock/**`, `**/sample*/**` |
| BizRules | `config/bizrules/`, `config/gwrules/` |

6. **Build the full inventory.** List every file, grouped by subsystem. Present to the user for review before proceeding. Do NOT start deleting until the inventory is reviewed.

### Phase R-1: Remove References and Consumers

Process one LOB at a time. For EACH symbol being removed (constant, typecode, entity field, display key):

1. **Grep the entire codebase** for the symbol name using start-boundary pattern.
2. **List every consumer found.** Include file path, line number, and usage context.
3. **Remove/update every consumer first.** When a reference is in dead code that only existed for this LOB, remove the entire block. When a reference is in shared code, remove only the LOB-specific branch/case.
4. **When removing an if-guard or case branch**, verify the surrounding syntax is still valid — check for orphaned braces `{` `}`, dangling else, or empty switch blocks. Read the file after editing to confirm it parses.
5. **When removing display key references**, remove the reference entirely — do NOT rename it to a different key (this caused 2 compile errors from CA7 and GL7 where keys were renamed to nonexistent targets).
6. **After all consumers are removed**, delete the definition itself.
7. **Grep again** to verify zero references remain.

### Phase R-2: Remove Definitions and Configurations

After all consumers are cleaned (Phase R-1 complete):

1. **Delete LOB entity files** (`.eti`, `.etx`) and corresponding typelist files (`.tti`, `.ttx`, `.tix`).
2. **Remove systable registrations** from `systables.xml`, `systables.CUST.xml`, and ALL variants. Then delete the data files.
3. **Remove rule files** (`.gr`) AND their entries from parent `order.txt` files. Missing either one causes `ClassNotFoundException` at startup.
4. **Remove plugin registrations** from `.gwp` files.
5. **Remove messaging destination entries** from `messaging-config.xml`.
6. **Remove scheduler entries** from `scheduler-config.xml`.
7. **Remove event rule entries** from entity `.etx` event blocks.
8. **Remove display keys** from ALL properties files — main, locale variants, schema display, productmodel display.
9. **Remove LOB product model** directories and their display name files (`.en`, `.es`, etc.).
10. **Remove LOB PCF files** and any wizard step/menu entries pointing to them.
11. **Remove LOB Gosu templates** (`.gst` files) — these reference entity properties at render time and WILL crash at runtime if the entities are gone. The Crime section in `VeriskPrometrixDocument_Ext.gst` was 147 lines of dead template code that would have crashed.
12. **Remove LOB test/mock files** from `gtest/` and mock directories.
13. **Remove smoketest entries** from `config/smoketests/typelists.properties` and related files.

### Phase R-3: Handle Known Fragile Files

These files aggregate content from multiple LOBs and break easily:

| File | Why It's Fragile | Required Action |
|---|---|---|
| `UnderwriterEvaluatorTriggeringPoint_Ext.gs` | Aggregates typecodes from ALL LOBs in one `addAll` block. One bad typecode cascades `ErrorType` to the entire array. Broken 3 separate times across WCM, CU, BP7. | Read fresh before EVERY LOB removal. After editing, verify EVERY remaining typecode exists in `RuleContextDefinitionKey.ttx`. |
| `EntityMetadataUtil.gs` | References constants from multiple LOBs. | After removing any constant, grep this file specifically. |
| `CommonConstants_Ext.gs` | Central constant repository — LOB-specific constants mixed with shared ones. | Remove only constants confirmed unused after Phase R-1 consumer removal. |
| `display.properties` (main) | Thousands of keys, LOB-specific mixed with shared. | Match on key name prefix before `=` sign. Only remove keys clearly LOB-specific (contain LOB prefix or `_Ext` LOB product names). |
| `productmodel.display.properties` | Can contain 1,000+ keys per LOB (BP7 had 1,310 QuestionSet keys, GL7 had 839). | Sweep with start-boundary grep on LOB prefix. These bulk keys are commonly missed. |
| REST API schema files (`*.schema.json`) | May reference LOB-specific typelists or properties. | Grep schema files for LOB types — `CUPolicyType` was missed in `policyperiod_ext-1.0.schema.json`. |

### Phase R-4: Exhaustive Verification Sweep (MANDATORY — DO NOT SKIP)

This phase exists because the user manually found 24+ categories of remnants that the agent missed. EVERY one of these checks is mandatory:

1. **Full codebase grep with start-boundary pattern** for EVERY LOB prefix being removed. Use patterns like `\bBP7`, `\bWCM`, `\bCU[^sS]` (careful with short prefixes — `\bCU` matches `CUSTOM`, `CURRENT`, etc.). For short LOB codes (CP, CU, GL, WC), add context-aware filtering.
2. **Verify ZERO hits in these specific files** (all were missed in past removals):
   - `config/locale/typelist.properties` — LOB rule context display keys (34 keys for all LOBs missed)
   - `config/locale/typelist_en_US.properties` — `LOBName_UM.*` entries
   - `config/locale/typelist_yy.properties` — test locale entries
   - `config/locale/typelist_es.properties` — Spanish translations
   - `config/locale/typelist_es_ES.properties` — Spain translations
   - `config/locale/display_en_US.properties` — LOB data flow mask keys (WCM had 15 keys)
   - `config/locale/display_yy.properties` — test locale keys
   - `config/locale/schema.display.properties` — LOB schema display keys
   - `config/locale/schema.display_en_US_EDG.properties` — Edge keys
   - `config/locale/schema.display_en_US_EDG_PH.properties` — Edge PH keys
   - `config/smoketests/typelists.properties` — smoketest references
   - `config/resources/ResourceContextMap.xml` — resource context typecodes
   - `config/extensions/typelist/` — ALL `.ttx` files for typecodes mentioning removed LOB
   - `config/xml/config.xml` — LOB-specific parameters (e.g., `EnableCPDTOParallelRating`)
   - `gsrc/**/*.gst` — Gosu templates with dead property references
   - `gtest/**` — dead test files referencing removed LOBs
   - `**/mock/**` and `**/sample*/**` — dead sample/mock data
3. **Check for entire LOB entity files still on disk.** `WCClassAdjustment_Ext.eti` was found still sitting on disk after WCM removal was declared "complete."
4. **Check display name (`.en`) files** in `config/extensions/displayname/` — `TerritoryCode.en` was missed for BP7.
5. **Check REST API schema files** under `integration/` for LOB-specific type references.
6. **Report all remaining hits to the user** with file path, line number, and match context. Do NOT silently skip any hit.

### Phase R-5: Compile and Server Gates

1. **Delete H2 database** — `tmp/pc.mv.db` and `tmp/pc.lock.db`. Stale H2 cache caused 3 startup failures.
2. **Run `gwb clean`** to clear codegen cache.
3. **Run `gwb codegen`** — must succeed with zero errors.
4. **Run `gwb compile`** — must succeed with zero errors. If errors found, classify each:
   - Consumer referencing deleted symbol → remove the consumer (return to Phase R-1)
   - Syntax error from incomplete edit → fix the orphaned syntax
   - Platform file error → restore from git, do NOT modify platform files
5. **Start the server** — must reach MULTIUSER run level. If startup errors:
   - `FileNotFoundException` → check systable registration files
   - `ClassNotFoundException` → check rule `order.txt` entries
   - `Invalid typelist` → check API schema files
   - H2 errors → delete H2 and restart
6. **Run final grep sweep** after all errors are fixed — new fixes may have introduced new orphans.
7. **Only declare LOB removal complete when**: compile is clean, server starts, AND the final grep sweep shows zero LOB references in customer-owned files.

### False Positive Handling

Short LOB prefixes generate false positives. Handle these explicitly:

| LOB Prefix | Known False Positives | How to Filter |
|---|---|---|
| CP | `CPClassCode`, `CPPolicy`, `CPPolicyBuildingIterative` (all platform CP types, not CPP) | Verify each match — is it `CP7`/`CPP` LOB code or platform Commercial Property? |
| CU | `CUSTOM`, `CURRENT`, `CURSOR`, `CUSTRenewalProcessImpl` | Use `\bCU[^a-z]` or grep for LOB-specific identifiers (`CULine`, `CUPolicyType`, `CUCoverage`) |
| GL | `GLOBAL`, `GL7` is specific but `GL` alone matches many things | Always use `GL7` or `\bGL7` as the primary pattern |
| WC | `WCM` is specific, but `WC7` also exists in platform typelists | Verify platform vs customer origin for each hit |
| CR | `CREATE`, `CREDENTIAL`, `CRITERIA` | Use `CR7` or LOB-specific identifiers (`CRLine`, `CrimeCoverage`) |

### Known Fragile File: UnderwriterEvaluatorTriggeringPoint_Ext.gs

This file was broken 3 times across 3 different LOB removals. Special handling:
1. Read the file fresh before EVERY LOB removal pass
2. After editing, verify EVERY remaining typecode exists in `RuleContextDefinitionKey.ttx`
3. If Gosu reports `ArrayList<ErrorType>` cascade errors, only 1-2 entries are actually bad — the cascading error makes ALL entries appear broken
4. Do NOT trust prior LOB removal passes left this file correct

## Patch/Delivery

### Creating the Patch
1. Generate from the Target OOTB baseline: `git diff -- modules/configuration`
2. Exclude binary files (PNG, JPG, GIF, fonts) — patches can't handle them
3. Exclude generated files (deploy/css/gen, deploy/js/gen) — client regenerates with `gwb compile`
4. SVG files ARE text and CAN go in the patch
5. The `deploy/` folder is a build output — source lives in `webresources/`

### Known Patch Issues
- **IntelliJ "File is read-only"** — Studio's internal VCS protection, not OS file permissions. No reliable workaround in Studio.
- **"Patch does not apply"** — Line ending mismatches between source and target. Use `git apply --ignore-whitespace` or deliver as zip instead.
- **Binary files** — Use `git diff --binary` for command-line `git apply`. Studio cannot handle binary diffs.

### Delivery Options
1. **Git branch** — client pulls from repo (cleanest)
2. **Patch file** — `git apply` on command line (works if same OOTB version)
3. **Zip of configuration** — extract and replace `modules/configuration` (simplest for non-technical clients, but overwrites client's own customizations)
4. **Delta zip** — new files copy directly, modified files need manual merge (safest for clients with existing customizations)
