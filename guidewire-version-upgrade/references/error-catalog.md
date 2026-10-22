# Error Catalog — Guidewire Version Upgrade

---

## Section A: LOB Removal — Compile Errors

### A1. Removed definition but left consumer (MOST COMMON — 10+ occurrences per LOB)

Every LOB removal hit this. The agent deletes a constant, typecode, or entity field, but consumers in other files still reference it.

| Symbol removed | Consumer left behind | File | LOB |
|---|---|---|---|
| `BOP_ELIGIBLE` typecode | `TC_BOP_ELIGIBLE` | `AccountHelper_Ext.gs` | BP7 |
| `NON_EDITABLE_COVTERMS_FIELDS_PRTL` | use in Gosu | `CovTermEnhancementDgc.gsx` | BP7 |
| `UMB_PRODUCTCODE_EXT` | reference in PCF | `SubmissionWizard_PolicyInfoDV.pcf` | CU |
| WCM constants | `WCM_EACHACCIDENTLIMIT_KEY` | `EntityMetadataUtil.gs` | WCM |
| BP7/WCM/CU typecodes | `TC_BP7MANUSCRIPTENDORSEMENT_EXT`, etc. | `UnderwriterEvaluatorTriggeringPoint_Ext.gs` | BP7/WCM/CU |

**Prevention:** Grep entire codebase for the symbol BEFORE deleting. Remove all consumers first. Grep again after.

### A2. Orphaned syntax from incomplete edit

Agent removed an if-guard or case branch but left orphaned braces `{` `}` or dangling else.

| File | Error | LOB |
|---|---|---|
| `SpecialCasesClauseLoadGenerator.gs` | Orphaned `{` and `}` braces after if-guard removed | WCM |

**Prevention:** After removing any control flow guard, read the surrounding code. Verify matching braces. Read the file after editing.

### A3. Display key renamed to nonexistent target instead of removed

Agent renamed a dead LOB display key reference to a different (also nonexistent) key, instead of removing the dead reference entirely.

| File | Error | LOB |
|---|---|---|
| `PolicyContactRolePanelSet.PolicyAddlInsured.pcf` | Renamed CA7 display key to nonexistent key | CA7 |
| `PolicyPeriodValidation_Ext.gs` | Renamed GL7 display key to nonexistent key | GL7 |

**Prevention:** When a display key reference points to a removed LOB, REMOVE the reference entirely. Never rename to another key unless you verify the target key exists.

### A4. Cascading ErrorType in UnderwriterEvaluatorTriggeringPoint_Ext.gs

When ONE typecode in the `addAll` block doesn't exist, Gosu types the entire array as `ArrayList<ErrorType>` and reports errors on ALL entries — even valid ones.

Broken 3 separate times: WCM removal, CU removal, BP7 removal.

**Prevention:** Read file fresh before every LOB pass. After edit, verify every remaining typecode exists in `RuleContextDefinitionKey.ttx`. Don't trust prior passes.

---

## Section B: LOB Removal — Server Startup Errors

### B1. FileNotFoundException — systable file deleted but registration kept

| File deleted | Registration left in | LOB |
|---|---|---|
| `wc_decile_dividend_map.xml` | `systables.xml` | WCM |
| `employment_practices_liability_excluded_class_ext.xml` | `systables.CUST.xml` | CU |

**Prevention:** Check `systables.xml`, `systables.CUST.xml`, and ALL `systables.*.xml` variants.

### B2. ClassNotFoundException — rule .gr deleted but order.txt entry kept

| Rule deleted | order.txt left in | LOB |
|---|---|---|
| `DividendComp_Ext.gr` | parent directory `order.txt` | WCM |

**Prevention:** When deleting any `.gr` file, always remove its entry from the parent `order.txt`.

### B3. Invalid typelist in REST API schema

| Type reference | Schema file | LOB |
|---|---|---|
| `CUPolicyType` | `policyperiod_ext-1.0.schema.json` | CU |
| `coverageNameCU_Ext` | same file | CU |

**Prevention:** Grep all `.schema.json` files under `integration/` for LOB type references.

### B4. False positive on hash ID deleted valid entry

| What was deleted | Why | LOB |
|---|---|---|
| Restaurant Section 2 QuestionChoice ("Night Club") | Auto-generated hash contained `bp7` as substring | BP7 |

**Prevention:** When grepping properties files, match on key name (before `=` sign) only, never the full line or hash portion.

### B5. BizRules bootstrap file references removed LOB

| Error | File | LOB |
|---|---|---|
| `BP7BusinessOwnersLine` BizRules import failure | Pre-existing gwrules bootstrap | BP7 |

**Prevention:** Check `config/bizrules/` and `config/gwrules/` for LOB references.

### B6. H2 cached stale state

3 startup failures from H2 database caching deleted entity schemas and systable checksums.

**Prevention:** Delete `tmp/pc.mv.db` and `tmp/pc.lock.db` after ANY entity, typelist, or systable change.

---

## Section C: LOB Removal — Missed Remnants (Found by User, Not Agent)

These are items the agent declared "complete" but the user found via manual IntelliJ searches. Organized by file type.

### C1. Properties files — bulk display keys left behind

| File | Keys left | LOB | Count |
|---|---|---|---|
| `productmodel.display.properties` | BP7 QuestionSet pre-qual keys | BP7 | 1,310 keys |
| `productmodel.display.properties` | GL7 QuestionSet pre-qual keys | GL7 | 839 keys |
| `display.properties` | CU Underlying Policy GL display keys | CU/GL7 | 58 keys |
| `display.properties` | CA7 validation deductibles/drivers/fleet keys | CA7 | 3 keys |
| `typelist.properties` | CUPolicyType display keys | CU | 10 keys |
| `typelist.properties` | LOB rule context display keys (all LOBs) | ALL | 34 keys |
| `typelist.properties` | `APDCloudProduct.CommercialUmb` + `InstalledPolicyLine.CU` | CU | 2 keys |
| `display_en_US.properties` | `WC7CoveredEmployee` data flow mask keys | WCM | 15 keys |
| `display_yy.properties` | `WC7CoveredEmployee` test locale keys | WCM | 15 keys |
| `schema.display.properties` | CU Underlying Policy schema keys | CU | multiple |
| `schema.display_en_US_EDG.properties` | CU Underlying Policy Edge keys | CU | multiple |
| `schema.display_en_US_EDG_PH.properties` | CU Underlying Policy Edge PH keys | CU | multiple |

### C2. Locale variant properties files — entire files not searched

| File | What was left | LOB |
|---|---|---|
| `typelist_en_US.properties` | `LOBName_UM.CommercialProperty` + `WorkersComp` | CP7/WCM |
| `typelist_yy.properties` | same entries | CP7/WCM |
| `typelist_es.properties` | `ClassCodeType.CR` Spanish translation | CR7 |
| `typelist_es_ES.properties` | `ClassCodeType.CR` Spain translation | CR7 |

### C3. Configuration files missed

| File | What was left | LOB |
|---|---|---|
| `config.xml` | `EnableCPDTOParallelRating` parameter | CP7 |
| `ResourceContext.ttx` | "Guidewire WC" resource context typecode | WCM |
| `PropFacCoverageBucket_Ext.tti` | "Commercial Property" typecode | CP7 |
| `smoketests/typelists.properties` | Smoketest references to removed typecodes | CPP |

### C4. Code files left on disk

| File | What it is | LOB |
|---|---|---|
| `WCClassAdjustment_Ext.eti` | Entire WCM entity file still on disk | WCM |
| `CPRequirements.gs` (gtest) | Entire CP7 rating test file | CP7 |
| `SmallSampleCommunityData.gs` | Contains "Jillian Losttime Workers Comp" sample data | WCM |
| `VeriskPrometrixDocument_Ext.gst` | ~147 lines of Crime template code referencing deleted properties | CR7 |

### C5. Displayname files missed

| File | What was left | LOB |
|---|---|---|
| `TerritoryCode.en` | BP7 displayname entry referencing deleted display key | BP7 |

---

## Section D: Migration Phase Compile Errors (WS-1 through WS-6)

### D1. Platform code transplanted
`gw.lob.common` platform classes were copied — these are internalized into Target JARs. Creates endless transitive dependency chains. Remove and use Target's extension points.

### D2. Non-LOB shared framework over-copied
Bulk-copying entire directories brings files for other LOBs. Trim to only what LOB code directly imports.

### D3. StructuredLogger subcategory missing
Source had custom subcategories (TEST, RATEFLOW, PCA, CONFIG_UI). Replace with existing Target subcategories (CONFIG, INTEGRATION).

### D4. Missing override keyword
Target compiler is stricter. Add `override` to methods implementing interfaces.

### D5. ArrayList to array conversion
Target WSDL methods expect arrays. Add `.toTypedArray()`.

### D6. Removed Description property
Target entity API changed. Remove the property access.

### D7. sync() signature changed
Target takes different number of arguments. Adapt the call.

### D8. Static vs instance access
Target requires static access for event constants (e.g., `Group.GROUPADDED_EVENT` not `group.GROUPADDED_EVENT`).

---

## Section E: Runtime Errors During Hardening (WS-7)

### E1. StandAloneBillingSystemPlugin NPE
`InvoiceItemType.Installment.GosuValue` returns null. Replace with string literal `"installment"`. Same for `ChargeCategory.Tax.GosuValue` → `"tax"` and `ChargeCategory.Premium.GosuValue` → `"premium"`.

### E2. Rate table multiplicand is null
UW company names in rate table don't match. Add correct UW companies with matching names.

### E3. UWCompany code unique constraint
UWCompanyCode is a typekey, not varchar. Add typecodes to the typelist extension.

### E4. Missing question sets
Coverage patterns referenced question sets that weren't migrated. Either copy them or remove the references.

### E5. DeductibleType mismatch
Coverage patterns referenced base `DeductibleType` instead of LOB-specific type. Fix the typelist reference.

### E6. ScriptParameters NPE
Wrong type or duplicate parameter. Fix type to `varchar`, remove duplicates.

### E7. Coverage fields not auto-filling
PCF PostOnChange wired to OOTB `APDRulesHelper` instead of custom `APDRulesHelper_Ext`. Wire to the custom helper with 3-argument `onChange(coverable, jobWizardHelper, term)`.

### E8. Required field markers missing
PCF required attribute using OOTB `term.Pattern.isRequiredForCoverable(coverable)`. Replace with custom `CovTermRequirednessInputSetHelper_Ext.isRequired(term)`.

### E9. Additional coverage fields showing too many fields
`CovTermDirectInputSetHelper` missing visibility logic. Replace with Source version containing `IM_NON_VISIBLE_COVTERMS_EXT` list.

### E10. Rate routine not found for coverage
Rate book doesn't have mapping for that coverage. Check rate book version — may need different version or need to activate Stage rate books.

### E11. Payment plans not loading
Rate books imported as Stage but `IRatingPlugin.gwp` has `RatingLevel=Approved`. Either change to `Stage` or set rate book XMLs to `Active` status.

### E12. Manuscript endorsement NPE
Direct coverage reference null when no endorsement selected. Use `CoverageCategoryInputSet` pattern with null guards.

### E13. Stale WSDL compiled classes
After modifying and restoring WSDL/XSD files, `idea-gclasses` cache retains old compiled classes. Delete `plugins/Gosu/idea-gclasses/wsi/` directory.
