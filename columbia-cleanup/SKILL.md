---
name: columbia-cleanup
description: Controlled, two-phase workflow for finding and removing/renaming client-specific references and implementation traces (Columbia, CIG, AMBest, NAICO, and their variants) in a Guidewire/insurance codebase, with a mandatory human approval gate before any code changes. Only trigger when the user explicitly invokes this skill by name or clearly asks to run "the Columbia cleanup," "the client cleanup skill," or a cleanup plan/analysis for these specific client references — do NOT trigger just because the conversation mentions Columbia, CIG, AMBest, NAICO, branding, refactoring, or reference removal in passing. Delegates repository discovery and reference tracing to the efficient-codebase-search skill when available, but owns all domain judgment: relevance, classification (Remove/Rename/Retain/Escalate), plan approval, execution, and validation.
---

# Columbia Cleanup

A two-phase workflow for removing or neutralizing client-specific references (Columbia, CIG,
AMBest, NAICO, and their variants) from a codebase, with a hard approval gate between "here's
what I found and propose" and "here's what I changed."

## Explicit invocation only

This skill does not self-trigger. Only proceed when the user has clearly asked for this specific
workflow — e.g. "run the Columbia cleanup," "use the cleanup skill on this module," "find the
client-specific references and build a cleanup plan." A passing mention of Columbia, CIG,
branding, or refactoring in unrelated conversation is not an invocation.

## Responsibility boundary

This skill is the domain authority. It decides *what* to search for, *what areas matter*, *what
findings mean*, *what to do about them*, and — importantly — *when discovery is complete enough to
plan* and *when another search pass is needed*. It never delegates those judgments.

Repository mechanics — where things live in this specific repo, how to search them without
burning context, tracing references and fallout, and running efficient repeated/verification
searches — go to the **efficient-codebase-search** skill when it's available. Invoke it explicitly
(`Skill("efficient-codebase-search")` or by describing the search intent as a delegated request)
rather than re-implementing search logic here. If that skill is unavailable, fall back to the same
discipline yourself: inspect repository structure before assuming a layout, search narrow-first,
reduce before inspecting, and only read what you need.

```
Cleanup skill:  "What are we looking for? What areas matter? Is discovery
                 converged? Is it relevant? Remove/Rename/Retain/Escalate?
                 Is the plan approved? Execute and validate."
                        │
                        ▼
Search skill:   "Where do those areas exist in THIS repo? How do I search
                 them efficiently? What references the target? Did this
                 repeated pass turn up anything materially new?"
```

The search skill provides mechanics and a convergence signal; it never decides how many passes to
run or what a finding means. Never let its output stand in as a finding — a search hit becomes a
finding only after this skill's relevance review and classification. Conversely, the cleanup skill
does not re-implement search mechanics; it *drives* the search skill in a controlled loop.

## Token Efficiency

These rules minimize token consumption without sacrificing correctness. The highest-cost
mistakes are: spawning one agent per file, re-reading files to "verify," repeating skill
instructions in agent prompts, and running verification after every phase instead of once.

- **Run `gwb` yourself in the background, filtered. This is the default and cheapest option.**
  This applies when verifying the cleanup (codegen, compile, runServer).
  - **How:** send output to a log file, then read back only the error lines. Example:
    `gwb compile > cleanup_compile.log 2>&1` (with `run_in_background`), then
    `grep -nE "error:|FAILED|Caused by|nonexistent|No type|No property|Could not resolve" cleanup_compile.log | head -150`.
  - **Time:** about 6 minutes per codegen or compile. Don't poll; you're notified when it
    finishes. The user is fine with the wait.
  - **Why it's cheaper:** an error batch costs a few dozen lines instead of hundreds of lines of
    Gradle stack trace. Never read or request the full log. If the user runs builds, ask for
    the filtered error lines only.
  - **runServer:** run it in the background to a log and watch for the ready line or
    ERROR/Exception with Monitor or an `until grep` loop. Read only the matched lines plus
    context. It takes a bit more back-and-forth than compile.
  - **Verify the repo root** (`ls gwb.bat modules/configuration`) before building.
- **Use efficient-codebase-search for ALL discovery.** Never spawn multiple search agents
  manually. One delegated search request with all targets and priorities replaces N separate
  grep agents.
- **Bulk property/config cleanup with sed, not agents.** Removing 500+ display keys, typelist
  entries, or properties lines is mechanical — use `sed` or `grep -v` patterns in Bash. Agent
  reasoning is wasted on line-by-line deletion.
- **One agent call per finding, not per file.** If finding F-003 requires edits in 4 files,
  send one agent call listing all 4 files and all edits. Never spawn 4 agents.
- **Don't spawn verification agents.** Run grep verification directly in Bash. A grep command
  costs ~100 tokens; an agent call to run the same grep costs ~2000+.
- **Skip reading platform-owned files.** If a search hit is in `gsrc/gw/` without `_Ext` in
  the name, classify it as platform and skip it. Don't read the file to confirm.
- **Reference codebase: one bulk comparison.** When a cleansed sibling codebase is available,
  run one directory-tree diff (`diff -rq`) at the start to identify all divergent files. Don't
  do per-file checks throughout discovery.
- **Compact agent prompts.** Don't repeat the search inventory, Iron Rules, or lesson-learned
  content in agent task prompts. The agent system prompt already has them. Prompts should
  contain: target identifiers, file list, action, exit criteria. Nothing else.
- **Don't re-read files to verify edits.** The Edit tool confirms success. Only re-read when
  there's a structural risk (XML well-formedness, JSON syntax, brace balance).
- **Single verification pass.** Run ONE comprehensive verification grep after all Phase 2
  execution is complete — not after each individual finding is addressed.
- **Parallel agents for independent work only.** Max 3-4 concurrent agents, each touching
  completely different files. Never parallelize agents that share any file.

## Effort Level Guidance

Claude cannot change its own effort level — **tell the user** when to switch (`/effort`).
Default to **medium**; ask for **high** only where reasoning quality changes the outcome.

| Step | Effort | Why |
|---|---|---|
| Session setup, loading inventory, writing artifacts | Medium | Mechanical |
| Discovery passes (Step 3) | Medium | Search skill does the mechanics |
| **Relevance review + classification (Steps 4, 7)** | **High** | Domain judgment — past misclassifications (AMBest, NAICO treaties, ratebook BookCode) caused multiple rework rounds |
| Reference/fallout analysis (Step 5) | Medium (High if dependency chain is unclear) | Mostly grep evidence |
| Phase 2 bulk edits (sed, renames, display keys) | Medium | Mechanical |
| **Integration rewrites (e.g. replacing JAR-backed API clients)** | **High** | Wrong approach corrupted JARs and cost several build cycles |
| **Compile/startup error that fails a second fix attempt** | **High** | Signals a non-obvious root cause |
| Verification sweep, Phase 2 report | Medium | Mechanical |

Prompt the user at each transition, e.g.: "Next step is classification — consider `/effort high`
for this step, then back to medium."

## Hard rule

**No code, config, or content modification happens before the user explicitly approves the
cleanup plan.** Discovery, analysis, artifact writing, and plan revision may proceed freely once
this skill is invoked — execution may not begin until approval is explicit and current (a
re-revised plan needs re-approval, not assumed carry-over consent).

---

## Session setup (do this first, before searching)

1. **Create a visible task list** (via the task tools) covering the whole workflow — the Phase 1
   steps below through the Phase 2 report — and keep statuses current as you go, so the user can
   see progress at any moment. This also guards against the failure mode where execution quietly
   stalls mid-Phase-1: the task list makes an unfinished step visible.
2. **Derive a run ID** once and reuse it all session — a timestamp like
   `columbia-cleanup-20260910-1432` (e.g. from `date`). Runs on the same repo must not collide.
3. **Create the reports directory** at the repository root:
   `reports/columbia-cleanup-<run-id>/`. All three artifacts are persisted here as `.md` files
   (see `references/templates.md`). Capture the repo name, branch, and module/working dir now —
   they go in the shared metadata block at the top of every artifact.

## Don't stop prematurely

Phase 1 is meant to run to completion on its own. Continue through the applicable steps without
pausing to check in — the only legitimate reasons to stop before the approval gate are: a genuine
hard-stop condition, information you truly cannot resolve or infer that blocks further work, or the
approval gate itself. "I've done some searching" is not a stopping point; discovery convergence,
relevance review, classification, and writing the two Phase 1 artifacts all still have to happen
first. If you find yourself wanting to stop, check the task list: if Phase 1 tasks remain and no
hard-stop applies, keep going.

---

## Phase 1 — Discover and Plan

### Step 1: Load the search inventory

Read `references/search-inventory.md` fresh each run — don't rely on memory of a prior
conversation, since the user may have edited it. It holds the primary targets (Columbia, CIG,
AMBest, NAICO), known variants, variant-generation patterns (separators, prefixes/suffixes,
compounds, case, file names, packages), and ambiguity warnings for overloaded terms like `CIG`.

If the user adds or removes a target for this run, treat that as a session-scoped addition unless
they ask you to update the inventory file itself.

### Step 2: Define domain-specific search priorities

State which kinds of areas matter for this run, without assuming a fixed repository layout.
Typical priorities for this codebase's domain:

- **Source code** — classes, interfaces, enums, methods, properties, constants, helpers,
  services, mappers, factories, plugins, integrations, package/namespace names, file names
- **Configuration and product model** — XML, config files, typelists, display keys, seed data,
  DB mappings, product definitions, form patterns, rate books, rating artifacts, policy
  identifiers, PCF files, UI configuration
- **User-facing values** — labels, dropdown values, tooltips, error messages, page titles, form
  names, company names, legacy labels
- **Developer-facing content** — comments, client-attribution notes, README/setup docs, test
  data, generated docs, developer metadata

These are priorities, not a boundary. Say so explicitly when handing off to search: unless the
user has restricted scope, the search should also cover whatever else repository discovery
surfaces as relevant (e.g. a custom module or integration module not on this list).

### Step 3: Discovery convergence loop (owned by this skill)

The central lesson from testing: a single discovery pass is not reliably complete — related
occurrences get missed, and valid candidates surface after discovery is assumed done. So discovery
is a **bounded loop this skill controls**, not a one-shot handoff:

1. **First pass.** Hand efficient-codebase-search a clear request: the targets/variants from the
   inventory, the priorities above, any explicit exclusions, and the evidence you need back
   (occurrences, definitions, references, fallout). Ask for compact results — occurrence locations
   and reference/dependency evidence, not raw file dumps. Let it determine the actual repo layout
   and where priorities map to.
2. **Review the returned set** and run relevance review (Step 4) on it so the follow-up pass can
   be *targeted*.
3. **Follow-up pass.** Ask the search skill for a limited pass that reuses its existing working
   record and looks specifically for candidates the previous pass didn't surface — in particular,
   **all related occurrences of confirmed findings** (other files touching the same identifier,
   config keys paired with a found label, sibling artifacts, references it flagged but didn't
   inspect). Because it reuses the record, it should report only *materially new* findings rather
   than re-processing unchanged files.
4. **Converge.** If the follow-up produced materially new valid candidates, fold them in and run
   one more bounded pass. Stop when a pass yields no materially new valid findings, or when you hit
   a small safety cap (**at most ~3–4 passes**) — whichever comes first. Record how many passes ran
   and whether the final one was clean; that becomes the convergence signal in the Search Report.

The search skill supplies the "did this pass find anything new?" signal; **this skill decides
whether to loop again and when discovery is done.** Don't ask the search skill to loop until it
decides cleanup is complete — that's a cleanup-workflow decision.

### Step 4: Relevance review

This is a domain judgment call this skill owns — never accept a search hit as a finding without
it. For each candidate:

1. Determine whether the occurrence is genuinely client-specific, using the surrounding
   identifier, class, configuration, label, or business context — not the matched text alone.
2. Identify and set aside unrelated uses (e.g. `CIG` as an unrelated acronym, `AMBest` as a
   generic industry reference rather than Columbia-specific usage — see the ambiguity warnings in
   the inventory file). Record what you set aside so the Search Report shows it was considered.
3. When a match is genuinely ambiguous and context doesn't resolve it, don't guess — defer it for
   investigation (more context, a targeted follow-up search, or asking the user) rather than
   silently dropping it or silently including it.

Only matches that survive this review become confirmed findings. **Assign each confirmed finding a
stable Finding ID (`F-001`, `F-002`, …) now, and use that same ID everywhere for the rest of the
session** — Search Report, Cleanup Plan, Phase 2 report. The ID is permanent identity; it never
changes when the classification changes (see `references/templates.md`).

### Step 5: Reference and fallout analysis

For every finding that might be renamed or removed, get enough dependency evidence to reason about
fallout before classifying it: direct references, referencing files, config/UI/product-model
relationships, tests, inheritance/implementation relationships, related identifiers. Request this
from efficient-codebase-search's reference-tracing and fallout-discovery steps — it labels
evidence as directly confirmed, likely related, or uncertain; carry those labels into your own
reasoning rather than treating everything as equally certain. Fallout evidence also feeds the
follow-up discovery passes in Step 3 (a reference the search flagged but didn't inspect is exactly
the kind of related occurrence a follow-up pass should chase).

### Step 6: Implicit client-specific logic (only when warranted)

Keyword search won't surface behavior that's client-specific in *logic* rather than *naming* —
custom business rules, validation, ordering, workflows, specialized structures with no
Columbia/CIG/AMBest/NAICO text anywhere in them. Investigate this only when one of the following
is true — don't default to a broad semantic sweep of the repo:

- Explicit search findings point at related custom behavior worth a closer look
- You have requirements knowledge suggesting a specific area is likely client-specific
- Fallout analysis surfaced suspicious related logic
- The user asks for broader investigation

When you do expand, still narrow-first: follow the lead from what triggered it rather than opening
unrelated areas "just in case."

### Step 7: Classify each confirmed finding

- **Remove** — client-specific component, branding, identifier, data, or behavior with no
  reusable purpose. Action: remove it, accounting for dependencies.
- **Rename / Neutralize** — functionality is reusable but its naming exposes client-specific
  ownership. Action: propose a neutral replacement name (this is always a proposal — the user
  approves or edits it, since there's no fixed neutral-naming convention assumed) and update
  affected references. **When proposing replacement identifiers — especially public-ids, keys, or
  codes in systable XML, typelists, or similar config — check for existing identifiers in the same
  file/scope that would collide.** Guidewire systable files combine a base file and overlay files
  into a single public-id namespace (e.g. `underwriting_companies.xml` + `underwriting_companies.
  IM.xml` share one namespace); a duplicate public-id across the combined file set is a fatal
  startup error ("Duplicate public ID"). Before proposing a replacement, read the full set of
  existing public-ids and choose one that doesn't collide. Prefer identifiers that are both neutral
  and distinguishable from the OOTB numbering sequence (e.g. `uwc:IM1` rather than `uwc:10` when
  OOTB uses `uwc:1`–`uwc:9`).
- **Retain** — the match is unrelated, required, part of a shared standard, or has valid reusable
  meaning. Document *why* it stays; don't just mark it retained.
- **Escalate** — ownership, functional impact, or business intent is unclear, or removal/renaming
  risks introducing uncertain behavior. Do not touch it without further review.

When in doubt between Retain and Escalate, prefer Escalate — Retain should be reserved for cases
you can actually justify, not a default for "not sure." Classification is a decision *separate*
from the Finding ID; changing it later never changes the ID.

### Step 8: Write the Search Report artifact

Write `01-search-report.md` to the reports directory using the Search Report template in
`references/templates.md`: shared metadata block, at-a-glance summary (targets, areas searched,
areas excluded, candidates reviewed, relevant findings, passes performed, whether the final pass
was clean, discovery status), the findings table with Finding IDs, what was set aside, what was
deferred, and the discovery pass log. This is the evidence record the plan is built on.

### Step 9: Write the Cleanup Plan artifact

Write `02-cleanup-plan.md` using the Cleanup Plan template: metadata block (based on
`01-search-report.md`), at-a-glance summary (counts per classification, open decisions, approval
status = Pending), and findings grouped by classification — each row carrying its Finding ID and
tracing back to the Search Report. Include only reviewed findings, never a dump of raw matches.
Each row: what/where, why relevant, what references it, what could be affected, classification,
recommended action, proposed replacement where applicable, and required validation.

### Step 10: Approval gate (interactive, explicit)

Present the plan and stop. Use an **interactive approval mechanism** (the AskUserQuestion tool)
so the decision is explicit rather than inferred from prose — offer clear options such as *Approve
as-is*, *Request changes*, and *Investigate further / run more searches*. The user may approve,
change classifications or actions, request deeper investigation or additional searches,
add/remove/exclude findings, approve or edit proposed replacement names, or ask for ambiguous
cases to be investigated further.

When the user changes anything before approving, **record the change in the plan's "User changes
before approval" section** (e.g. `F-004: Rename → Retain`), update the affected rows and the
approval-status summary, re-persist `02-cleanup-plan.md`, and present again. This preserves the
distinction between what was originally recommended and what was actually approved — which the
Phase 2 report depends on. Revise and re-present until the user gives explicit approval to execute.
Don't infer approval from silence, from approval of an earlier version of the plan, or from the
user moving on to a different topic.

---

## Phase 2 — Execute (only after explicit approval)

Use the approved plan as the execution source of truth. Perform only what it approves, referring
to findings by their stable IDs:

- **Remove** — remove the approved item, accounting for the dependencies identified during fallout
  analysis.
- **Rename / Neutralize** — apply the approved (or user-edited) replacement name and update every
  dependent reference identified during fallout analysis, consistently.
- **Retain** — make no modification.
- **Escalate** — make no modification unless and until it receives its own explicit approval.

**Scope control:** don't perform unrelated refactoring or opportunistic cleanup, and don't modify
anything newly discovered during execution that wasn't in the approved plan — surface it instead
(see Remaining matches below).

### Validation and post-execution verification

After executing the approved changes:

1. **Check for identifier collisions first.** For any renamed public-id, key, or code: grep the
   target file (and its overlay/companion files that share the same namespace) for the new
   identifier to confirm it doesn't already exist. Guidewire systable files are merged at startup
   — `underwriting_companies.xml` and `underwriting_companies.<LOB>.xml` share one public-id
   namespace, so a collision across files is fatal. This check must happen before attempting to
   start the server.
2. Rebuild/compile affected modules where possible.
3. Check for broken references caused by the approved changes; fix them when they're a clear,
   direct consequence of an approved action (e.g. a rename that missed one call site) — this is
   still within the approved scope, not new opportunistic work.
4. Run available automated tests, and any targeted tests for the specific functionality touched.
4. **Final verification search.** Re-run the configured inventory via efficient-codebase-search,
   asking it to **reuse the working record from discovery** so it can compare against
   already-known, changed, retained, and deferred items and report only what's genuinely new.
   Confirm approved targets were actually removed/neutralized as expected. Re-check file names and
   identifiers too, not just content — renames/removals can leave stale file or package names.

### Remaining matches

Categorize what the verification search still finds:

- **Intentional retain** — already covered by a Retain decision (reference the Finding ID)
- **Unrelated / false positive** — doesn't need action
- **Requires further investigation** — genuinely unclear, same as an Escalate
- **Missed cleanup item** — looks like it should have been covered but wasn't

Do not act on newly discovered items automatically, even ones that look like obvious misses —
return them to the workflow for classification and a fresh plan/approval cycle. The approval gate
applies to every round of changes, not just the first:
**new findings → revised plan → approval → execution → verification.**

### Step: Write the Phase 2 report artifact

Automatically persist `03-cleanup-report.md` after execution and validation, using the Phase 2
template. This is the entry point for reviewing the session, so it leads with the summary:

- **Session outcome** near the top — overall status (Complete / Complete with deferred items /
  Follow-up required / Partial or failed), counts of approved vs. executed vs. failed/skipped,
  retained, escalated, build/test/verification status, and number of newly discovered candidates.
- **Approved plan vs. actual result** table — one row per Finding ID showing approved decision,
  actual result, and status — so it's immediately visible whether Phase 2 followed Phase 1.
- Detailed evidence below: actions performed, files modified/removed, references updated,
  validation results, remaining matches, newly discovered candidates, and recommended follow-up.

The report should stand on its own — a reviewer should understand exactly what happened and why
without the conversation history, tracing any finding across all three artifacts by its ID.

---

## Lessons learned (accumulated from prior sessions)

These are hard-won patterns from previous cleanup runs. Apply them proactively — don't repeat
past mistakes.

### 1. Generated files, mock data, and WSDLs are in scope

The verification search must cover **all** file types, not just source and config:

- **`generated/`** — auto-generated Java files from entity definitions (`.eti`/`.etx`). These
  contain the same entity/column names and will show up in searches. If you delete or modify the
  source entity definition, also delete the corresponding generated Java files — they'll
  regenerate on the next build. Don't leave stale generated files with old names.
- **Mock/test data XML and JSON** — integration mock response files (e.g.
  `SB_response-mock.xml`, `response-mock.xml`, `answers-mock.json`) often contain client-specific
  names, carrier IDs, and email addresses embedded deep in large payloads. These are easy to miss
  in initial search passes because they may be single-line JSON or very long XML. Search
  explicitly for targets in `*mock*` and `*test*` files.
- **WSDL files** — third-party schema definitions (e.g. `RISKXMLWSDL.wsdl`) may contain
  client-specific field names as XML schema elements. If the code that reads those fields is
  removed, the schema elements should also be removed.
- **Generated WSDL copies** — if a WSDL exists in both `gsrc/` and `generated/`, clean both.

### 2. Do NOT auto-retain AMBest as "industry standard"

Prior cleanup of IM-Independent proved that AMBest entities, systables, hazard-grade logic, and
integration model classes are **client-specific customizations**, not reusable industry
capabilities. The correct action is full removal:
- Delete the `AMBestHazard_Ext.eti` entity definition entirely
- Delete the `AM_Best_Hazard_Grade.xml` systable and its registration
- Delete business rules (`.gwrules`)
- Remove `AMBestNumber` columns from entity extensions
- Remove AMBest model classes, mapping functions, and import statements
- Remove AMBest elements from WSDLs
- Remove AMBest field-meta entries and display keys
- Delete all generated files derived from removed entities

### 3. Entity column removal is acceptable for cleanup

Earlier sessions were too conservative about entity/column `name=` attributes, treating them as
immutable DB schema. In practice, for independent/clean codebases (not migrating live data),
removing entity columns and entire entity definitions is the correct approach — matching what
IM-Independent did. Only preserve column names when there is confirmed live data that depends
on them.

### 4. Cross-reference with a cleansed sibling codebase when available

When another LOB's codebase has already been cleansed (e.g. IM-Independent), use it as the
reference standard. Search for the same targets in the cleansed codebase — if a reference is
**absent** there, it should be removed here too. This catches classification errors like the
AMBest "Retain" mistake.

### 5. Seed data and treaty files must be cleaned — never auto-retain

Treaty/reinsurance seed data files (e.g. `1_AllProgramsAndTreaties.xml`) contain client-specific
program names, contract references, and comments with NAICO/Columbia references. Prior sessions
incorrectly classified these as "Retain" assuming they were immutable legal contract names. In
practice, the cleansed IM-Independent codebase has **no** treaty file or NAICO references at all.
Replace client names in treaty data (e.g. "NAICO Chandler" → "Carrier Chandler",
"NAICO/Chandler" → "Carrier/Chandler") or remove the file entirely. Also update any test helpers
that check for client-specific book code prefixes (e.g. `startsWith("NAICO ")` → match the new
naming convention).

### 6. Single-line JSON/XML needs explicit handling

Large single-line files (mock JSON responses, concatenated XML) get truncated in grep output and
are easy to miss. For mock/test data files, extract all distinct matching strings
(`grep -oP` / pattern extraction) rather than relying on line-level output.

### 7. gwgen.cig.pc JAR removal — replace with Gosu classes, NEVER binary-repackage

The `gwgen.cig.pc.*.jar` files are generated API client libraries with the CIG package namespace
baked into compiled bytecode. **Never** attempt to rename the package inside JARs using `sed` or
any binary replacement tool — it corrupts class files and Gosu's classloader rejects alternative
package names due to namespace collisions.

**Proven approach (from successfully cleaned PC-N codebase):**

**Self-contained modules** (only called by test pages, no external business logic callers):
Delete entirely — source, tests, JARs, test page PCFs, and LocationRef entries in parent PCFs.
- NCCI (`gsrc/cust/pc/integration/ncci/`) — self-contained, delete all
- Verisk BUR (`gsrc/cust/pc/integration/verisk/bur/`) — self-contained, delete all
- Verisk LossCost (`gsrc/cust/pc/integration/verisk/losscost/`) — self-contained, delete all
  (BUR and LossCost are cross-dependent — always remove together)

**DMS (Webdocs)** — deeply integrated, 6+ external callers, plugin-registered:
1. Create simple Gosu model classes in `cust.pc.integration.dms.model`:
   - `DMSDocumentType_Ext.gs` — fields: `Id : String`, `Title : String`, `Fields : List<DMSDocumentTypeField_Ext>`
   - `DMSDocumentTypeField_Ext.gs` — fields: `Id : String`, `Title : String`, `IsRequired : boolean`
   - `DMSFolder_Ext.gs` — fields: `Id : String`, `Title : String`
2. Rewrite `WebdocsDMSRestClient.gs`: replace `ApiConfig.buildAPI(FoldersApi).foldersGet()`
   and `ApiConfig.buildAPI(DocumentTypesApi).documentTypesGet()` with direct HTTP calls using
   `HttpGet` + `JsonObject.parse()`, returning the new model classes instead of JAR types.
   Replace all return types and internal references (`Folder` → `DMSFolder_Ext`,
   `DocumentType` → `DMSDocumentType_Ext`, `DocumentTypeField` → `DMSDocumentTypeField_Ext`).
3. Update `WebdocsDMSMetadataCache.gs`: change import and type references from
   `gwgen.cig.pc.integration.dms.model.DocumentType` to `cust.pc.integration.dms.model.DMSDocumentType_Ext`.
4. Delete the JAR: `gwgen.cig.pc.integration.dms-1.0.jar`

**PriceDigests VIN** — called by `JobApiExtHandler.gs`:
1. Replace `VINIntegAdapter_Ext.gs` with a stub that returns an empty list. Keep the class name
   and `getVehicleDetails(vin : String)` signature so `JobApiExtHandler.gs` compiles.
2. Create `VINResponseDTO_Ext.gs` with all the vehicle detail fields (ModelID, ModelYear,
   ModelName, Brand, etc.) as the return type.
3. Delete `VINRestClient_Ext.gs`, `VINMockService_Ext.gs`, and the `mock/` directory.
4. Delete the JAR: `gwgen.cig.pc.pricedigests-1.0.0.jar`

**When a reference codebase is available**, copy the cleaned files directly from it rather than
rewriting from scratch — this avoids subtle differences in method signatures or JSON parsing.

### 8. Ratebook BookCode/BookName/BookDesc must NOT be renamed

Ratebook internal metadata (`BookCode`, `BookName`, `BookDesc`, and their L10N `Value` elements)
are **runtime identifiers**, not branding. Guidewire generates availability classes from the
`BookCode` at startup, and the `*-availability-config.xml` references those generated class paths.
Changing the `BookCode` from e.g. `Columbia Crime CW` to `Crime CW` breaks the availability
config's class references, causing `ClassNotFoundException` at runtime when creating submissions.

**What to do:**
- Ratebook **file names** can be renamed (cosmetic, no runtime impact)
- Ratebook **BookCode, BookName, BookDesc** inside the XML must keep their original values
- The availability config XML (`cr7-availability-config.xml` etc.) depends on these values
- This is the same principle as NAIC codes on UW companies — the identifier stays, only
  human-readable labels and file names change

### 9. UW companies — ALWAYS rename, all three, everywhere, to `UW Carrier <NAIC>`

The three underwriting companies are Columbia Insurance Group's own carriers and are always
client-specific. **Classify as Rename — never Retain or Escalate.** Use the AAIS-reference naming:

| public-id | Code | NAIC | Original name | Rename to |
|---|---|---|---|---|
| `uwc:CNI` | `0807_19640_Ext` | 0807-19640 | Columbia National Insurance Company | `UW Carrier 19640` |
| `uwc:CMI` | `0807_40371_Ext` | 0807-40371 | Columbia Mutual Insurance Company | `UW Carrier 40371` |
| `uwc:ACI` | `0807_35629_Ext` | 0807-35629 | Association Casualty Insurance Company | `UW Carrier 35629` |

- **Association Casualty is a CIG company too** — it contains no "Columbia"/"CIG" text, so a
  keyword-only review misses it. A past run renamed only two of the three, leaving the UW Company
  dropdown with mixed naming.
- **Keep public-ids, typecodes, and NAIC codes unchanged** — only `Name`/`ParentName`/`name`/`desc`.
- **Rename in ALL of these together, in one pass** (sed is fine — plain text):
  1. `config/resources/systables/underwriting_companies.xml` — `<Name>`, `<ParentName>`
  2. `config/extensions/typelist/UWCompanyCode.ttx` — `name`, `desc`
  3. **Every ratebook** under `config/content/cust-ratebooks/**` — `<str1>` values in UW-company-keyed
     rate tables (e.g. `CrimeLCM_Ext`, `FidelityForgeryLCM_Ext`)
  4. Any `TypeKey.UWCompanyCode.*` overrides in `config/locale/typelist*.properties`
- **Why the ratebooks are mandatory:** `LCMPreemptionHandler_Ext.gs` passes
  `UWCompanyCode.DisplayName` into an **exact-match** rate table lookup. Renaming the typelist but
  not the ratebook `str1` rows silently breaks the company-specific loss cost multiplier — no
  error, just wrong premium. A past run made exactly this mistake.
- **Verify:** grep for all three original names AND the half-renamed forms (`National Insurance
  Company`, `Mutual Insurance Company`) → zero hits; count `<str1>UW Carrier` rows per company
  matches the original row count.
