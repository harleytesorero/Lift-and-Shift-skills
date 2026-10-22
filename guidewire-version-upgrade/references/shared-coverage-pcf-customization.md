# Shared Coverage PCF Customization (APD Lines)

## When to Apply

Check if the Source+LOB has **custom coverage helpers** that extend the OOTB shared coverage PCFs. Look for:
- `gsrc/cust/pcf/coverage/CovTermRequirednessInputSetHelper_Ext.gs` — custom required-field logic
- `gsrc/cust/pcf/coverage/APDRulesHelper_Ext.gs` — custom onChange with auto-fill/dependency logic
- `gsrc/cust/pcf/coverage/CovTermTypkeyInputSetHelper.gs` — custom typekey visibility/editability
- `gsrc/gw/pcf/coverage/CovTermDirectInputSetHelper.gs` — custom direct term visibility/editability
- `gsrc/gw/pcf/coverage/CovTermOptionInputSetHelper.gs` — custom option term visibility/editability/onChange
- LOB-specific APD rules helpers (e.g., `IMAPDRulesHelper_Ext.gs`, `GLAPDRulesHelper_Ext.gs`)

If any of these exist, the LOB uses a **custom APD coverage framework** where:
- Fields marked as required in the product model need the custom helper to show as required in the UI
- Coverage term values auto-fill based on other selections (e.g., ExcessLimit = TotalLimit - BaseInclLimit)
- Some coverage terms are conditionally visible/editable (e.g., hidden BaseIncl/Excess fields)
- Selecting a value in one field triggers dependent fields to appear or populate

Without these PCF changes, the OOTB shared coverage PCFs use the platform's default behavior, which ignores all custom requiredness, auto-fill, and conditional visibility logic.

## What to Migrate

### 1. Copy Custom Helper Classes (WS-3: Gosu)

From Source+LOB, copy:
```
gsrc/cust/pcf/coverage/CovTermRequirednessInputSetHelper_Ext.gs
gsrc/cust/pcf/coverage/APDRulesHelper_Ext.gs
gsrc/cust/pcf/coverage/CovTermTypkeyInputSetHelper.gs
gsrc/gw/pcf/coverage/CovTermDirectInputSetHelper.gs
gsrc/gw/pcf/coverage/CovTermOptionInputSetHelper.gs
```

Also copy any LOB-specific helpers referenced by `APDRulesHelper_Ext`:
```
gsrc/cust/lob/<lob>/IMAPDRulesHelper_Ext.gs   (or equivalent for the LOB)
```

And the shared coverage enhancement:
```
gsrc/cust/lob/common/CovTermEnhancement_Ext.gsx
```

### 2. Modify Shared Coverage PCFs (WS-7: Shared/Cross-LOB)

These are OOTB shared PCFs under `config/web/pcf/shared/coverage/`. They serve ALL lines, so changes must be conditional — only trigger custom behavior for APD lines.

#### 2a. CovTermInputSet PCFs — Three Attribute Changes Each

Apply to ALL CovTermInputSet mode PCFs that have coverage term inputs:
- `CovTermDirectInputSet.pcf`
- `CovTermInputSet.Option.pcf`
- `CovTermInputSet.Typekey.pcf`
- `CovTermInputSet.shorttext.pcf`
- `CovTermInputSet.longtext.pcf` (if customer-created)
- Any other CovTermInputSet mode PCFs

**Change 1 — `required` attribute:**
```xml
<!-- OOTB -->
required="term.Pattern.Required"

<!-- Custom -->
required="cust.pcf.coverage.CovTermRequirednessInputSetHelper_Ext.isRequired(term)"
```

**Change 2 — `onChange` attribute (inside PostOnChange):**
```xml
<!-- OOTB (2-arg, does NOT trigger auto-fill) -->
onChange="gw.web.rules.APDRulesHelper.onChange(coverable, jobWizardHelper)"

<!-- Custom (3-arg, triggers LOB-specific auto-fill via IMAPDRulesHelper_Ext etc.) -->
onChange="cust.pcf.coverage.APDRulesHelper_Ext.onChange(coverable, jobWizardHelper, term)"
```

**Change 3 — `disablePostOnEnter` attribute (inside PostOnChange):**
```xml
<!-- OOTB -->
disablePostOnEnter="not gw.web.rules.APDRulesHelper.hasDependents(coverable, term.Pattern)"

<!-- Custom (conditional: APD lines use custom helper, non-APD lines get false) -->
disablePostOnEnter="(coverable.PolicyLine.APDLine) ?cust.pcf.coverage.APDRulesHelper_Ext.hasDependents(coverable, term):false"
```

#### 2b. CovTermDirectInputSet.pcf — Additional visible/editable

```xml
<!-- Add to the TextInput element -->
visible="gw.pcf.coverage.CovTermDirectInputSetHelper.isVisible_Ext(term)"
editable="gw.pcf.coverage.CovTermDirectInputSetHelper.isEditable_Ext(term)"
```

#### 2c. CovTermInputSet.Option.pcf — Additional visible/editable/onChange

```xml
<!-- Add to the RangeInput element -->
visible="gw.pcf.coverage.CovTermOptionInputSetHelper.isVisible_Ext(term as gw.api.domain.covterm.CovTerm)"
editable="gw.pcf.coverage.CovTermOptionInputSetHelper.isEditable_Ext(term as gw.api.domain.covterm.OptionCovTerm, openForEdit and editable)"

<!-- onChange is conditional: APD lines use APDRulesHelper_Ext, non-APD use CovTermOptionInputSetHelper -->
onChange="if (coverable.PolicyLine.APDLine) {cust.pcf.coverage.APDRulesHelper_Ext.onChange(coverable, jobWizardHelper,term)} else {gw.pcf.coverage.CovTermOptionInputSetHelper.onChange_Ext(term as gw.api.domain.covterm.OptionCovTerm)}"
```

#### 2d. CovTermInputSet.Typekey.pcf — Additional visible/editable

```xml
<!-- Add to the RangeInput element -->
visible="cust.pcf.coverage.CovTermTypkeyInputSetHelper.isVisible_Ext(term as gw.api.domain.covterm.TypekeyCovTerm)"
editable="cust.pcf.coverage.CovTermTypkeyInputSetHelper.isEditable_Ext(...)"
```

#### 2e. CoverageInputSet.default.pcf

Copy from Source+LOB if it has custom changes. Typical customizations:
- `disablePostOnEnter` uses `APDRulesHelper_Ext.hasDependents` instead of OOTB
- Checkbox visibility changed for APD coverage patterns

### 3. CoverageCategoryInputSet Pattern (LOB-specific PCFs)

Some LOB PCFs use `CoverageCategoryInputSet` instead of `CoverageInputSet` for specific coverables. This is the pattern for rendering coverages on a coverable that has a fixed set of coverage patterns:

```xml
<Variable
  initialValue="coverableEntity != null and coverableEntity.SomeCov != null ? new gw.api.productmodel.ClausePattern[]{coverableEntity.SomeCov.Pattern} : new gw.api.productmodel.ClausePattern[]{}"
  name="covPatterns"
  recalculateOnRefresh="true"
  type="gw.api.productmodel.ClausePattern[]"/>

<InputSetRef
  def="CoverageCategoryInputSet(covPatterns, coverableEntity, openForEdit, jobWizardHelper)"
  id="CovPatterns"/>
```

This is needed when a coverable's PCF panel must show coverage terms but the coverable may be null (e.g., manuscript endorsements, optional coverage parts).

## What These Changes Enable

| Feature | OOTB Behavior | With Custom Helpers |
|---------|---------------|---------------------|
| Required fields | Only `term.Pattern.Required` (static) | Dynamic requiredness per product model rules |
| Auto-fill | No auto-fill | ExcessLimit = TotalLimit - BaseInclLimit, etc. |
| Conditional visibility | All terms visible | BaseIncl/Excess fields hidden unless applicable |
| Conditional editability | All terms editable | Read-only fields based on product model rules |
| Dependent fields | No dependency chain | Selecting Risk Group shows dependent fields |
| onChange triggers | 2-arg (no term context) | 3-arg (term-aware, triggers LOB-specific logic) |

## Why These Are Safe for Other LOBs

All changes are conditional on `coverable.PolicyLine.APDLine`:
- APD lines (like IM) → custom helpers fire
- Non-APD lines → falls through to OOTB behavior or returns safe defaults

The custom helpers themselves also guard internally:
- `APDRulesHelper_Ext.onChange` checks if the line has a registered LOB helper
- `CovTermRequirednessInputSetHelper_Ext.isRequired` falls back to `term.Pattern.Required` for unknown lines
- `isVisible_Ext` / `isEditable_Ext` return `true` by default (show everything) for non-customized lines

## Common Symptoms When These Changes Are Missing

| Symptom | Root Cause |
|---------|------------|
| Required fields not showing red asterisks | `required` still uses `term.Pattern.Required` instead of custom helper |
| Auto-fill not calculating (e.g., Installation Floater) | `onChange` uses 2-arg OOTB version, not 3-arg custom version |
| Fields that should be hidden are visible | Missing `visible` attribute with custom `isVisible_Ext` |
| Selecting a dropdown value doesn't trigger dependent fields | `disablePostOnEnter` not using custom `hasDependents` |
| MTCC/MTCO Risk Group selection doesn't show new fields | Same as above — `onChange` and `disablePostOnEnter` need custom helpers |
| Coverage terms all editable when some should be read-only | Missing `editable` attribute with custom `isEditable_Ext` |

## Verification

After server restart (PCFs need compiled expression classes rebuilt):

1. **Required fields** — Open a coverage with required terms. Red asterisks should appear.
2. **Auto-fill** — On Installation Floater, change TotalLimit. ExcessLimit should auto-calculate.
3. **Conditional visibility** — BaseIncl/Excess fields should be hidden unless the coverage pattern enables them.
4. **Dependent fields** — On MTCC/MTCO, select a Risk Group value. New dependent fields should appear.
5. **Manuscript endorsement** — Open the Manuscript Endorsement tab. Coverage terms should render without NPE.
