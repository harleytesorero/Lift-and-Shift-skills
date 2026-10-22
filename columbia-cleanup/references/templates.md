# Templates

This file defines the three persisted artifacts of a cleanup session and the conventions shared
across them. The goal is that a **human reviewer** can open any one of these files on its own and
understand it without reconstructing the session from the conversation, while an **LLM** still has
the full evidence trail for deeper comparison. The artifacts should stand alone — an LLM should
enhance review, not compensate for reports that are hard to follow.

## Where reports go

Persist all three artifacts as `.md` files (not just inline output) under a per-run directory at
the repository root:

```
reports/columbia-cleanup-<run-id>/
├── 01-search-report.md
├── 02-cleanup-plan.md
└── 03-cleanup-report.md
```

- `<run-id>` must be unique enough that multiple runs on the same repo (even the same day) don't
  collide — use a timestamp, e.g. `columbia-cleanup-20260910-1432`. Derive it once at session
  start (e.g. from `date`) and reuse it for the whole session.
- The three files are numbered so the reading order is obvious: Search Report → Cleanup Plan →
  Phase 2 Report.
- No separate fourth "session summary" artifact is needed — the Phase 2 report is the entry point
  for reviewing a completed session; the other two remain as supporting evidence.

## Shared session metadata (top of every artifact)

Every artifact opens with the same metadata block so any one of them is self-locating:

```markdown
> **Session:** columbia-cleanup-<run-id>
> **Artifact:** 1 of 3 — Search Report
> **Date:** YYYY-MM-DD
> **Repository:** <repo name>   **Branch:** <branch>
> **Module / working dir:** <path>
> **Based on:** <previous artifact filename, or "—" for the Search Report>
```

Set `Artifact:` to `1 of 3 — Search Report`, `2 of 3 — Cleanup Plan`, or `3 of 3 — Phase 2 Report`
as appropriate. The Cleanup Plan is `Based on: 01-search-report.md`; the Phase 2 report is
`Based on: 02-cleanup-plan.md (approved)`.

## Stable Finding IDs

Assign each confirmed finding a stable ID — `F-001`, `F-002`, … — at the point it survives
relevance review, and **carry that same ID through the Search Report, Cleanup Plan, and Phase 2
report unchanged.** The ID is the finding's permanent identity; the classification is a separate,
mutable decision. Never encode the classification into the ID (no `R1`/`E1`/`N1`), because the
user may move a finding from Rename to Retain to Escalate during review and the identity must
survive that.

```
Search Report:   F-003 — UW company references (relevant, ambiguous-resolved)
Cleanup Plan:    F-003 — Escalate
Phase 2 Report:  F-003 — Not modified; escalation retained
```

---

## Artifact 1 — Search Report (`01-search-report.md`)

Answers: **what did the search find?** Written after discovery has converged (see the
discovery-convergence loop in SKILL.md) and relevance review is done.

```markdown
[shared session metadata block]

# Columbia Cleanup — Search Report

## At a glance
- **Search targets:** [list, or "configured inventory — see references/search-inventory.md, plus: …"]
- **Areas searched:** [priority areas + any additional areas repository discovery surfaced]
- **Areas excluded:** [with reason, or "none"]
- **Candidate matches reviewed:** N
- **Relevant findings:** N
- **Discovery passes performed:** N
- **Final pass produced new findings?** yes / no  ← convergence signal
- **Discovery status:** converged / stopped at safety limit (N passes)

## Findings
| ID | Finding (what / where) | Why relevant | Referenced by / fallout | Ambiguity notes |
|---|---|---|---|---|
| F-001 | … | … | … | — |
| F-002 | … | … | … | CIG match — confirmed entity context |

## Set aside (not findings)
Matches reviewed and deliberately excluded, so a reviewer sees they were considered, not missed:
| Match / location | Why not a finding |
|---|---|

## Deferred for investigation
Ambiguous matches context couldn't resolve — carried forward, not silently dropped:
| Match / location | What would resolve it |
|---|---|

## Discovery pass log
Brief record of each pass and what it added, so the convergence decision is auditable:
- Pass 1 — [areas/targets] — M candidates, K new relevant findings
- Pass 2 — [follow-up focus, e.g. "related occurrences of F-002 identifier"] — K new
- Pass 3 — no materially new findings → converged
```

---

## Artifact 2 — Cleanup Plan (`02-cleanup-plan.md`)

Answers: **what is being proposed, and what requires approval?** Findings only, grouped by
classification — never a dump of raw search hits.

```markdown
[shared session metadata block]

# Columbia Cleanup — Cleanup Plan

## At a glance
- **Remove:** n    **Rename / Neutralize:** n    **Retain:** n    **Escalate:** n
- **Open decisions:** n  (items needing a user call before execution)
- **Approval status:** Pending / Changes requested / Approved

## Remove
| ID | File / Area | Matched term | Referenced by / fallout | Recommended action | Validation needed |
|---|---|---|---|---|---|

## Rename / Neutralize
| ID | File / Area | Matched term | Referenced by / fallout | Proposed replacement (pending approval) | Validation needed |
|---|---|---|---|---|---|

## Retain
| ID | File / Area | Matched term | Why retained |
|---|---|---|---|

## Escalate
| ID | File / Area | Matched term | Why escalated | What would resolve it |
|---|---|---|---|---|

## Open questions for the user
- [anything needing a decision before execution]

## User changes before approval
Populated as the user revises the plan, so the Phase 2 report can distinguish the *original*
recommendation from the plan that was *actually approved*. Empty until the first revision.
- F-004: Rename → Retain (user decision)
- F-007: replacement name "GenericLocationMapper" → "LocationMapper" (user edit)
- F-012: added for removal at user request
```

Notes:
- "Referenced by / fallout" summarizes dependency evidence in a line or two — don't paste raw
  search output.
- "Proposed replacement" is always a proposal pending approval, never presented as decided (no
  fixed neutral-naming convention is assumed).
- Every row traces back to a Search Report finding by ID.

---

## Artifact 3 — Phase 2 Report (`03-cleanup-report.md`)

The main entry point for reviewing a completed session. A reviewer opening this first should
immediately see what was approved, what actually changed, whether execution succeeded, what
validation ran, and whether follow-up is needed. Detailed evidence stays below the summary.

```markdown
[shared session metadata block]

# Columbia Cleanup — Phase 2 Report

## Session outcome
- **Overall status:** Complete / Complete with deferred items / Follow-up required / Partial or failed
- **Approved changes:** n    **Executed successfully:** n    **Failed or skipped:** n
- **Retained:** n    **Escalated (deferred):** n
- **Build/compile:** pass / fail / n/a    **Tests:** pass / fail / n/a    **Verification search:** clean / new candidates found
- **Newly discovered candidates (not yet approved):** n

## Approved plan vs. actual result
| ID | Approved decision | Actual result | Status |
|---|---|---|---|
| F-001 | Rename | Renamed successfully | Complete |
| F-002 | Remove | Removed successfully | Complete |
| F-003 | Escalate | Not modified | As planned |
| F-004 | Rename | Could not execute — [reason] | Attention |

## Actions performed
### Removed
- F-00x: [item — brief description]
### Renamed / Neutralized
- F-00x: [old → new — files updated]
### Retained (no change)
- F-00x: [reason]
### Escalated / Skipped
- F-00x: [reason not acted on]

## Files modified
- [path — nature of change]
## Files removed
- [path]
## References updated
- F-00x: [files where references were updated as a consequence of the approved change]

## Validation
- **Build / compile:** [status; any errors + how resolved]
- **Tests:** [what ran, pass/fail, any targeted tests for affected functionality]
- **Post-cleanup verification search:** re-ran configured inventory against [areas], reusing the
  Search Report's working record. Remaining matches categorized below.

## Remaining matches
| Match / location | Category |
|---|---|
| … | Intentional retain (F-00x) |
| … | Unrelated / false positive |
| … | Requires further investigation |
| … | Missed cleanup item — NOT modified, needs approval |

## Newly discovered candidates
Items surfaced during verification that were not in the approved plan. **Not modified** — flagged
for a follow-up plan/approval cycle.
- [candidate — where — why it looks relevant]

## Recommended follow-up
- [...]
```
