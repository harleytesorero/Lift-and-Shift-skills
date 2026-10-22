# Search Inventory

This is the configured, editable list of client-specific search targets for the
`columbia-cleanup` skill. Read this file at the start of every run — don't rely on memory of it
from a prior conversation, since the user may have edited it.

Treat this file as the source of truth for **what to search for**. It does not decide relevance —
that judgment happens later, in the Relevance Review step of SKILL.md, using project context.

## Primary targets

- Columbia
- CIG
- AMBest
- NAICO

## Known variants

- AM Best
- Columbia Insurance
- Columbia Insurance Group
- Columbia Mutual
- Columbia National
- Association Casualty
- Columbia Plus
- CIG Legacy
- CIGLegacy
- CMI (acronym for Columbia Mutual Insurance)
- ACI (acronym for Association Casualty Insurance)
- NAIC codes `19640`, `40371`, `35629` / typecodes `0807_19640_Ext`, `0807_40371_Ext`,
  `0807_35629_Ext` — CIG's three UW carriers. Always Rename to `UW Carrier <NAIC>` across
  systables, UWCompanyCode.ttx, AND ratebook `<str1>` rows (see SKILL.md lesson 9).

## Variant patterns to generate per target

For each primary target and known variant above, also consider (don't blindly search every
permutation — generate the ones that plausibly occur in this codebase's conventions, informed by
what repository discovery turns up):

- **Separator variants** — with/without spaces, hyphens, underscores, dots, camelCase/PascalCase
  joins (`Columbia`, `Columbia_`, `Columbia-`, `ColumbiaCIG`, `CIG_Columbia`)
- **Prefix/suffix variants** — the term as a prefix or suffix on class/file/package names
  (`ColumbiaMapper`, `MapperColumbia`, `im_columbia_config`, `columbia.im.config`)
- **Compound identifiers** — the term combined with a domain word (`ColumbiaLocation`,
  `NAICORateBook`, `CIGTypelist`)
- **Case variants** — all-caps, all-lowercase, Title Case, camelCase, PascalCase as they'd appear
  in code identifiers vs. prose vs. labels
- **File-name occurrences** — the term appearing in a file or directory name rather than its
  contents
- **Package/namespace occurrences** — the term as a path segment or namespace component

## Ambiguity warnings — do not treat as automatic findings

Some entries in this inventory are short and generically meaningful outside this client context.
Flag matches for these as ambiguous and apply extra scrutiny in Relevance Review — do not classify
them as findings on the strength of a text match alone:

- **CIG** — could collide with unrelated abbreviations (e.g. generic acronyms in comments, other
  business terms). Confirm the surrounding class/config/label context actually refers to the
  Columbia Insurance Group entity before treating it as a finding.
- **AMBest / AM Best** — while AM Best is a third-party rating agency name, in practice the
  AMBest entities, systables, hazard-grade logic, and integration model classes in CIG codebases
  are client-specific customizations that should be **removed or renamed**, not retained. Prior
  cleanup of IM-Independent confirmed complete removal. Do NOT auto-retain AMBest references as
  "industry standard" — they are client-specific implementations that use AM Best data. Evaluate
  each match, but the default expectation is Remove/Rename, not Retain.
- **CMI** — a very short, generic-looking acronym that can collide with unrelated terms (e.g.
  other business/technical abbreviations, variable names). Confirm the surrounding
  class/config/label context actually refers to Columbia Mutual Insurance before treating it as
  a finding.
- **ACI** — likewise short and generic; collides easily with unrelated acronyms (e.g. "Access
  Control Item", other industry terms). Confirm the surrounding context actually refers to
  Association Casualty Insurance before treating it as a finding.

## Maintaining this list

The user may ask to add, remove, or adjust entries in this inventory over the course of a
cleanup engagement (e.g. after finding a new variant during search). Update this file directly
when that happens so future runs stay current.
