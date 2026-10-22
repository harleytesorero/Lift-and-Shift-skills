# LOB Removal Lessons — Guidewire Version Upgrade

These lessons were learned from removing 8 LOBs (CP7, CA7, WCM, GL7, CR7, CU, BP7, CPP) to isolate IM.
The user manually found 24+ categories of remnants and 10+ compile errors the agent missed. Every lesson below represents a real failure.

---

## Part 1: The 13 Hard-Won Rules

### 1. Always Remove ALL Consumers Before Removing a Definition
15+ compile errors caused by this — on EVERY LOB removal. Grep the ENTIRE codebase before deleting any symbol. Remove consumers first, delete definition second, grep again to verify zero references.

Specific failures:
- BP7: Removed `BOP_ELIGIBLE` typecode but left `TC_BOP_ELIGIBLE` in `AccountHelper_Ext.gs`
- BP7: Removed `NON_EDITABLE_COVTERMS_FIELDS_PRTL` from `CommonConstants_Ext.gs` but left use in `CovTermEnhancementDgc.gsx`
- WCM: Removed WCM constants but left `WCM_EACHACCIDENTLIMIT_KEY` in `EntityMetadataUtil.gs`
- CU: Removed `UMB_PRODUCTCODE_EXT` from constants but left reference in `SubmissionWizard_PolicyInfoDV.pcf`

### 2. Trace Feature Subsystems, Not Just Individual Files
20+ files missed by only looking at direct references. For every LOB feature, trace the full chain:
- Entity → batch processes → scheduler entries
- Entity → event rules → order.txt → messaging destinations
- Entity → plugin registries (.gwp)
- Entity → webservice endpoints (.wsdl, .wsc)
- Entity → display names, display keys (ALL locale variants)
- Entity → systable registrations (ALL systables*.xml files)
- Entity → Gosu templates (.gst) — runtime crash risk
- Entity → mock data files, sample data
- Entity → REST API schema files (.schema.json)

Entire subsystems missed:
- **WC Dividends** (WCM): 20+ files — entities, typelists, batch, PCFs, rules, messaging, plugins, display keys, webservice
- **NCCI Integration** (WCM): 7+ files — entity extension, typelists, OfficialID types, enhancement, logger, test
- **Crime Models/Verisk/ConvR** (CR7): 8 files — 3 model classes, 3 response model fields, 2 mapping methods + 1 Gosu template
- **EPLI Events & Systables** (CU): Events in PolicyPeriod.etx, systable data files, systables.CUST.xml registrations

### 3. Check ALL Systables Registration Files
3 FileNotFoundExceptions from missing this. Always check:
- `systables.xml`
- `systables.CUST.xml`
- Any other `systables.*.xml` variant

Specific failure: `employment_practices_liability_excluded_class_ext.xml` was deleted but its registration left in `systables.CUST.xml` — the agent didn't know this file existed.

### 4. Delete Rule order.txt Entries When Deleting .gr Files
1 ClassNotFoundException crash (`DividendComp_Ext`) from a leftover `order.txt` entry pointing to a deleted rule.

### 5. Delete H2 Database After Entity/Systable Changes
3 startup failures from stale H2 cache. Delete `tmp/pc.mv.db` and `tmp/pc.lock.db` after ANY entity or typelist change.

### 6. Never Modify Platform-Owned Files
45 files wrongly modified during removal, all had to be restored from git. Platform files include:
- `gw/webservice/*` API Javadoc (15 base GW webservice API files changed: SubmissionAPI, ImportPolicyAPI, etc.)
- `config/locale/typelist*.properties` translation files (for base typecodes)
- `config/locale/display_*.properties` translation files
- `config/metadata/*`
- Base `.eti`/`.tti`/`.tix` files
- `RICoverageGroupType` description strings in locale files

**Only clean customer-owned locations:** `cust/*`, `ext/*`, `config/extensions/`. For `display.properties` (main English), only remove keys clearly customer-added (contain `_Ext` or are LOB-specific product model keys).

### 7. Grep Patterns Must Use Start-Boundary Only
Use `\bWCM` NOT `\bWCM\b`. The word-boundary at the end misses compound identifiers:
- `WCMLocation`, `WCMLargeDeductible`, `WCMonopolistic`, `WCClassCode` — all missed by `\bwcm\b`

### 8. Display Key Grep Must Match Key Names, Not Hash IDs
Server startup failure: `grep -vi "bp7"` on productmodel.display.properties deleted a key whose auto-generated hash contained `bp7` as a substring. This removed a valid Restaurant Section 2 QuestionChoice ("Night Club"). Match on key name prefix before the `=` sign only.

### 9. UnderwriterEvaluatorTriggeringPoint_Ext.gs Is Fragile
This file was broken 3 times across WCM, CU, BP7 removals. It aggregates typecodes from ALL LOBs in one `addAll` block. When the first typecode fails to resolve, Gosu types the entire array as `ArrayList<ErrorType>` — every subsequent entry appears broken (cascading error). Only 1-2 entries are actually bad.

Required handling:
1. Read fresh before EVERY LOB removal
2. After editing, verify EVERY remaining typecode exists in `RuleContextDefinitionKey.ttx`
3. Do NOT trust prior LOB passes left it correct

### 10. Sequence Definition Deletions After Reference Updates
NEVER run definition-deleting agents in parallel with reference-renaming agents. One agent may delete a definition that another agent is trying to rename references to. Sequential execution only:
- Agent A (reference updater) runs first and completes
- Agent B (definition deleter) runs after

Specific failure: Display.properties agent bulk-deleted keys while Gosu agent renamed references to those keys → orphaned compile errors (GL7 validation display key, CA7 AutoDealer display key).

### 11. Check Gosu Templates (.gst) When Removing Model Properties
`VeriskPrometrixDocument_Ext.gst` had 147 lines of HTML template referencing deleted `responseExt.Crime.*` properties. These don't cause compile errors — they crash at RUNTIME when the template renders.

### 12. Check Plugin Registry (.gwp), Messaging Config, and Scheduler
3 missed registrations that caused startup failures:
- Plugin registry files (`*.gwp`)
- Messaging destination configurations
- Scheduler/batch process registrations

### 13. Verify Mock/Test Data After Model Changes
Dead mock files and dead GUnit tests left behind:
- `CPRequirements.gs` (gtest) — entire CP7 rating test file left on disk
- `SmallSampleCommunityData.gs` — "Jillian Losttime Workers Comp" sample data left behind
- `WCClassAdjustment_Ext.eti` — entire WCM entity still on disk after removal declared "complete"

---

## Part 2: Complete Catalog of User-Caught Remnants

These are ALL the issues the user found manually that the AI agent missed. Each one is a gap that the skill must cover.

### Compile Errors Found by User After `gwb compile`

| # | File | Error | LOB | Root Cause |
|---|------|-------|-----|------------|
| 1 | `UnderwriterEvaluatorTriggeringPoint_Ext.gs` | WCM typecodes not removed | WCM | Removed typecodes from typelist but didn't grep Gosu consumers |
| 2 | `UnderwriterEvaluatorTriggeringPoint_Ext.gs` | CU typecodes not removed | CU | Same file broken again — didn't learn from WCM |
| 3 | `UnderwriterEvaluatorTriggeringPoint_Ext.gs` | BP7 typecodes not removed | BP7 | Same file broken a THIRD time |
| 4 | `EntityMetadataUtil.gs` | Referenced deleted WCM constants | WCM | Deleted constants without grepping for consumers |
| 5 | `SpecialCasesClauseLoadGenerator.gs` | Orphaned `{` and `}` braces | WCM | Removed if-guard but left braces |
| 6 | `PolicyContactRolePanelSet.PolicyAddlInsured.pcf` | Renamed CA7 display key to nonexistent key | CA7 | Renamed key instead of removing dead reference |
| 7 | `PolicyPeriodValidation_Ext.gs` | Renamed GL7 display key to nonexistent key | GL7 | Same mistake — renamed instead of removing |
| 8 | `AccountHelper_Ext.gs` | Referenced deleted `TC_BOP_ELIGIBLE` | BP7 | Removed typecode, didn't grep for consumers |
| 9 | `CovTermEnhancementDgc.gsx` | Referenced deleted `NON_EDITABLE_COVTERMS_FIELDS_PRTL` | BP7 | Removed constant, didn't grep for consumers |
| 10 | `SubmissionWizard_PolicyInfoDV.pcf` | Referenced deleted `UMB_PRODUCTCODE_EXT` | CU | Removed constant, didn't grep for consumers |

### Server Startup Errors Found by User

| # | Error | LOB | Root Cause |
|---|-------|-----|------------|
| 1 | `CUPolicyType` invalid typelist in `policyperiod_ext-1.0.schema.json` | CU | Didn't clean REST API schema |
| 2 | `coverageNameCU_Ext` in same schema | CU | Same file, different property missed |
| 3 | Restaurant Section 2 QuestionChoice missing — "Night Club" hash contained `bp7` | BP7 | Grep false positive on hash ID |
| 4 | `wc_decile_dividend_map.xml` FileNotFoundException | WCM | Deleted systable file but left registration in `systables.xml` |
| 5 | `employment_practices_liability_excluded_class_ext.xml` FileNotFoundException | CU | Deleted file but left registration in `systables.CUST.xml` |
| 6 | `DividendComp_Ext` ClassNotFoundException | WCM | Deleted rule `.gr` files but left entry in `order.txt` |
| 7 | `BP7BusinessOwnersLine` BizRules import failure | BP7 | Pre-existing gwrules bootstrap file references removed LOB |

### Remnants Found by User During Manual Searches (24 Categories)

| # | File | What was found | LOB | Why the agent missed it |
|---|------|---------------|-----|------------------------|
| 1 | `display.properties` | CU Underlying Policy GL display keys (58 keys) | CU/GL7 | Shared keys not purely LOB-prefixed |
| 2 | `productmodel.display.properties` | BP7 QuestionSet pre-qual keys (1,310 keys) | BP7 | Bulk keys with product model hashes |
| 3 | `productmodel.display.properties` | GL7 QuestionSet pre-qual keys (839 keys) | GL7 | Same — hashed key names |
| 4 | `PropFacCoverageBucket_Ext.tti` | "Commercial Property" typecode still present | CP7 | Natural language name, not code prefix |
| 5 | `config.xml` | `EnableCPDTOParallelRating` CP7 parameter | CP7 | Config parameter, not in typical search paths |
| 6 | `SmallSampleCommunityData.gs` | "Jillian Losttime Workers Comp" sample data | WCM | Sample data file not in search scope |
| 7 | `display.properties` | CA7 validation deductibles/drivers/fleet keys (3 keys) | CA7 | Keys named after business concepts, not LOB prefix |
| 8 | `typelist.properties` | CUPolicyType display keys (10 keys) | CU | Typelist properties not swept |
| 9 | `typelist.properties` | LOB rule context display keys (34 keys for ALL LOBs) | ALL | Shared rule context keys for all removed LOBs |
| 10 | `typelist_en_US.properties` | `LOBName_UM.CommercialProperty` + `WorkersComp` | CP7/WCM | Locale variant not searched |
| 11 | `typelist_yy.properties` | `LOBName_UM.CommercialProperty` + `WorkersComp` | CP7/WCM | Test locale not searched |
| 12 | `schema.display.properties` | CU Underlying Policy schema keys | CU | Schema display file not searched |
| 13 | `schema.display_en_US_EDG.properties` | CU Underlying Policy Edge keys | CU | Edge locale not searched |
| 14 | `schema.display_en_US_EDG_PH.properties` | CU Underlying Policy Edge PH keys | CU | Edge PH locale not searched |
| 15 | `display_en_US.properties` | `WC7CoveredEmployee` data flow mask keys (15 keys) | WCM | EN_US locale variant not searched |
| 16 | `display_yy.properties` | `WC7CoveredEmployee` test locale keys (15 keys) | WCM | YY test locale not searched |
| 17 | `typelist.properties` | `APDCloudProduct.CommercialUmb` + `InstalledPolicyLine.CU` | CU | Typelist entries named after business concept |
| 18 | `typelist_es.properties` | `ClassCodeType.CR` Spanish translation | CR7 | Spanish locale not searched |
| 19 | `typelist_es_ES.properties` | `ClassCodeType.CR` Spain translation | CR7 | Spain locale not searched |
| 20 | `VeriskPrometrixDocument_Ext.gst` | Entire Crime section (~147 lines) referencing deleted `responseExt.Crime.*` | CR7 | .gst template files not in search scope |
| 21 | `CPRequirements.gs` (gtest) | Entire CP7 rating test file | CP7 | gtest directory not searched |
| 22 | `ResourceContext.ttx` | "Guidewire WC" resource context typecode | WCM | .ttx file with natural language description |
| 23 | `WCClassAdjustment_Ext.eti` | Entire WCM entity still on disk | WCM | Entity file somehow missed during deletion |
| 24 | `TerritoryCode.en` | BP7 displayname file referencing deleted display key | BP7 | .en displayname files not in search scope |

---

## Part 3: Parallel Agent Coordination Rules

1. **Never have one agent bulk-delete from a properties/config file while another agent renames references** in that same file
2. When a file has been edited by a prior removal pass, the **next pass must re-read the file fresh**
3. After all agents complete, **do a compile check before declaring the phase done**
4. Prefer **removing dead code/UI elements entirely** over renaming references to nonexistent keys
5. After removing a definition, **grep again** — even if you just grepped before removing consumers (new orphans can appear from parallel edits)

---

## Part 4: False Positive Handling

| LOB Prefix | Known False Positives | Mitigation |
|---|---|---|
| CPP/CP | `CPPolicy`, `CPPolicyBuildingIterative`, `CPClassCode` — all platform CP types, not CPP LOB | Verify each match is LOB code vs platform Commercial Property |
| CU | `CUSTOM`, `CURRENT`, `CURSOR`, `CUSTRenewalProcessImpl` | Use `\bCU[^a-z]` or grep specific LOB identifiers |
| GL | `GLOBAL`, many platform terms start with GL | Use `GL7` or `\bGL7` as primary pattern |
| WC | `WC7` exists in platform typelists | Verify platform vs customer origin |
| CR | `CREATE`, `CREDENTIAL`, `CRITERIA` | Use `CR7` or LOB-specific identifiers (`CRLine`, `CrimeCoverage`) |
| BP | `BPProcess`, other platform terms | Use `BP7` or `\bBP7` as primary pattern |

Additional: `smoketests/typelists.properties` is ALWAYS missed — it's in `config/smoketests/` which is not in any typical search path. Add it explicitly to every sweep.

---

## Part 5: Post-Removal Verification Checklist

After every LOB removal, before declaring done:

- [ ] Full grep sweep with start-boundary patterns across ALL mandatory search directories
- [ ] ZERO hits in all locale variant properties files (en_US, yy, es, es_ES, EDG, EDG_PH)
- [ ] ZERO hits in smoketests directory
- [ ] ZERO hits in REST API schema files
- [ ] ZERO LOB entity files remaining on disk
- [ ] ZERO LOB displayname (.en/.es) files remaining
- [ ] ZERO LOB .gst template files remaining
- [ ] ZERO LOB gtest files remaining
- [ ] `UnderwriterEvaluatorTriggeringPoint_Ext.gs` verified — all remaining typecodes exist
- [ ] `gwb codegen` passes
- [ ] `gwb compile` passes with zero errors
- [ ] H2 database deleted (`tmp/pc.mv.db`, `tmp/pc.lock.db`)
- [ ] Server starts to MULTIUSER run level
- [ ] Final grep sweep AFTER all fixes (fixes can introduce new orphans)
