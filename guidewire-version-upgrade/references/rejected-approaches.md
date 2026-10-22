# Rejected Approaches — Guidewire Version Upgrade

These approaches were tried and rejected before arriving at the delta extraction method.

## Approach 1: Git Merge with Orphan Branch

**What:** Create an orphan branch with Target content, merge the Source+LOB branch into it.

**Why it failed:** Caused thousands of false merge conflicts because of unrelated git histories. Git couldn't determine common ancestors between the orphan branch and the existing branch, so every file appeared as a conflict.

**Lesson:** Git merge only works when branches share a common ancestor. Orphan branches have no shared history.

## Approach 2: Incremental Diff-Patch

**What:** Generate diffs between each intermediate OOTB version pair (N→O, O→P, P→Q) and apply with `--3way`.

**Why it failed:** Overly complex. Required obtaining every intermediate OOTB version. Many intermediate patches conflicted because they touched the same files. The accumulated conflicts were harder to resolve than doing a direct migration.

**Lesson:** Going through intermediate versions is unnecessary. Extract the customer delta once and apply it to the target. The PDF approach of going straight Source→Target is simpler and more correct.

## Approach 3: Full Tree Copy (Source → Target)

**What:** Copy the entire Source+LOB `modules/configuration` over the Target OOTB.

**Why it was wrong:** Overwrites all Target OOTB improvements, security patches, API changes, and new features. Also brings non-LOB client code, other LOB remnants, and outdated platform files.

**Lesson:** The target OOTB must be the base. Never overwrite it.

## The Correct Approach: Delta Extraction

**(Source + LOB) minus (Source OOTB) = Customer delta**

Apply customer delta to Target OOTB, adapting for API differences.

This is the approach documented in the main SKILL.md.
