---
name: efficient-codebase-search
description: Domain-agnostic, token-efficient strategy for searching and investigating large codebases — repository discovery, mechanical search, result reduction, selective inspection, definition lookup, reference tracing, and dependency/fallout discovery. Use whenever the user or a calling workflow explicitly asks to "use efficient codebase search," "search these identifiers efficiently," "trace references for X," or delegates repository discovery / reference tracing to a search strategy. Does NOT trigger automatically just because code or a repo is being discussed — only on explicit invocation. This skill only finds and reports; it never decides what to do with findings (no renaming, deleting, refactoring, or classifying results as safe/unsafe) — that judgment belongs to the caller.
---

# Efficient Codebase Search

A reusable investigation strategy for large codebases. It answers **"how and where can I
efficiently find this, and what references it?"** — never **"what should I do with it?"**

The caller (a user or another workflow) supplies the search intent: what to look for, and
optionally where to look and what kind of answer is needed. This skill supplies the *mechanics*:
how to find it cheaply, reduce the noise, and report back just enough evidence for the caller to
reason about.

## Why this exists

Reading files into model context is the most expensive way to explore a codebase. Repository
search tools (`rg`, `grep`, `find`, git-aware search, IDE indexes) can scan an entire repo for a
fraction of the tokens a single large file costs to read. The strategy here is to push as much
work as possible onto those tools, and reserve model reasoning for the parts a tool genuinely
can't do: deciding what's worth a closer look, understanding relationships, and summarizing.

**Core loop:** Discover Repository → Search → Reduce → Inspect → Trace → Summarize

**Core principle:** Use repository and search tools for discovery. Use model context only for
information that requires reasoning.

## Scope boundary — read this first

This skill does not know what the search targets *mean*. It doesn't decide whether a match is a
false positive for domain reasons, whether an identifier should be renamed, or whether a
reference is safe to remove. It reports evidence; the caller applies domain judgment. The one
kind of filtering this skill *does* apply on its own is mechanical noise — build output,
generated code, vendored dependencies — because that's a search-mechanics concern, not a domain
one.

## Step 1 — Establish scope

Resolve scope in this priority order, and stop as soon as one applies:

1. **Explicit caller scope.** If the caller gives directories, file types, priorities, or
   exclusions, honor them. Treat "prioritize X" as "search X first, but also check the rest of
   the repo unless the caller says to *restrict* the search to X" — callers usually mean priority,
   not a hard boundary, unless they say so explicitly.
2. **Inferred repository scope.** If the caller gives a search target but no scope, run a
   lightweight structural inspection before searching (see below). Infer source/config/test/
   generated/build/dependency areas from that inspection.
3. **Clarification.** Only ask the caller when a scope decision is materially important and
   genuinely can't be inferred cheaply (e.g., the repo has multiple unrelated products and the
   target name is ambiguous across them). Don't ask for anything a directory listing or manifest
   file would answer.

### Lightweight repository inspection

When scope must be inferred, use cheap, shallow techniques — never start by opening source files:

- Shallow directory listings and depth-limited trees
- File-extension counts to spot the dominant languages/formats
- Build/project manifest discovery (e.g. `pom.xml`, `package.json`, `*.iml`, `build.gradle`,
  `.gwapp` / module descriptors) to find module boundaries
- Name-pattern detection for generated/build/vendor directories (`build/`, `dist/`, `gen/`,
  `node_modules/`, `target/`, `generated/`, etc.)

The goal is a rough map — module boundaries, where source/config/tests/generated code live — not
an understanding of what the code does.

## Step 2 — Mechanical search

Search with tools, not by reading. Use `rg`/`grep`/`find`/git-aware search/IDE search facilities.
Collect compact hits only: file path, line number, matched target, matching line, and — only when
it materially helps identification — a line or two of surrounding context. Don't open whole files
at this stage unless a file is already small and a single read is clearly cheaper than several
targeted searches against it.

Run separate searches per search type the caller asked for (file contents, file names, identifier
definitions, references) rather than one big fuzzy pass — each has a different tool shape and a
different noise profile.

## Step 3 — Reduce results

Before any deeper inspection, compress the raw hit list:

1. Group by file, then by identifier.
2. Deduplicate repeated matches; count occurrences; keep the line numbers.
3. Combine closely related results (e.g. multiple hits in the same method).
4. Apply caller-approved exclusions.
5. Strip obvious mechanical noise (build output, generated artifacts) when the caller allows it.

Turn a wall of hits like `Foo.gs:42`, `Foo.gs:51`, `Foo.gs:70`, `Foo.gs:81` into one line: `Foo.gs
— Example — 4 occurrences — lines 42,51,70,81`. See `references/reduction-and-batching.md` for
the full before/after example and the batching pattern for very large result sets (hundreds of
raw matches → tens of candidates worth inspecting).

Flag — don't resolve — domain ambiguity. A short, overloaded identifier like `CIG` might mean
several unrelated things; report occurrence/file counts and a note that it looks ambiguous, and
let the caller decide which occurrences matter.

## Step 4 — Selective inspection

Inspect only the candidates the reduction step marked as needing more context, and start with the
smallest useful unit — the matching lines, the enclosing method/class, the XML element, the config
block — before ever pulling in a whole file. Expand only when the current snippet can't answer the
question at hand. This is where most of the token budget for a search should go, and it should
only go to candidates that earned it in Step 3.

## Step 5 — Definitions and references (when requested)

When a hit looks like a reference to an identifier (class, method, config key, XML/JSON/YAML key,
UI component id, etc.), locate its definition mechanically the same way — search, don't browse.
Stay framework-neutral: only chase framework-specific constructs (e.g. a particular plugin
interface) when the caller names them: don't invent domain assumptions about what a "service" or
"plugin" looks like in an unfamiliar framework.

For reference tracing, follow: identifier → locate definition → search repo for usages → collect
and group reference locations → inspect only the references the caller actually needs. Don't open
every referencing file by default — report the compact reference set first (definition location,
occurrence count, reference locations) and inspect further only on request or when the caller's
question can't be answered without it. Output format and a worked example are in
`references/reference-tracing.md`.

## Step 6 — Dependency / fallout discovery (only when requested)

When the caller asks for fallout or dependency information, look for direct references, outgoing
dependencies, related config/UI/metadata/tests, inheritance and interface relationships. Fallout
findings are inherently less certain than a direct text match — a related config file might genuinely
govern the target, or might just happen to share a name — so label **every** such finding, not just
the ones that feel surprising:

- **Directly confirmed** — a mechanical search or definition lookup proves the relationship (e.g. the
  file literally imports/calls the target, or a manifest explicitly lists it).
- **Likely related** — naming, location, or structural pattern suggests a relationship, but nothing
  confirms it mechanically (e.g. a test file named after the target that doesn't actually import it).
- **Uncertain / requires deeper analysis** — worth flagging, not worth chasing further without caller
  sign-off.

For example, when reporting "related config" or "related tests" for a target, don't just state what
you found — tag it: "`sbt-common-contents.xml` lists the 3 callers but not the target itself —
directly confirmed gap" or "no test files found — directly confirmed absence of coverage." A finding
with no label attached is a sign it wasn't run through this step. Do not recursively walk the whole
dependency graph by default; expand only when the caller asks, or the evidence so far is insufficient
to answer their question.

## Progressive depth

Match effort to what's actually needed — don't start deep:

| Level | What happens |
|---|---|
| 0 — Repository discovery | Only when scope is missing. Cheap structural inspection, no source reading. |
| 1 — Discovery | Mechanical search only. Compact hits. |
| 2 — Reduction | Group, dedup, count, apply exclusions. |
| 3 — Targeted inspection | Read small sections of prioritized candidates. |
| 4 — Reference investigation | Definitions, usages, selected reference inspection. |
| 5 — Deep investigation | Broader semantic/dependency work — only when explicitly requested, relationships stay unclear after Level 4, hidden dependencies are suspected, or the caller flags this as high-impact. |

Never start at Level 5. Climb through the levels only as far as the caller's question requires.

## Working record

Keep a compact running record per target instead of re-reading files you've already searched:
target, definition location, relevant files, occurrence count, reference count, key references,
inspection status, notes. Reuse it for the rest of the investigation.

Also preserve, at the investigation level: which areas were searched, which exclusions were
applied, which matches became known findings, and which were deferred as ambiguous. This record is
what makes a later repeated or verification pass cheap — see below.

## Repeated and verification searches

Callers often run more than one pass over the same repository — a follow-up discovery pass, or a
post-change verification pass. When a request is clearly a repeat (the caller says so, or asks you
to reuse a prior search record), lean on the working record instead of starting from scratch:

- **Reuse the record.** Treat the preserved searched-areas / exclusions / known-findings /
  deferred-items list as your starting point rather than re-deriving scope and re-running every
  search from zero.
- **Distinguish known from new.** Separate results into matches already in the record (known
  findings, previously set-aside items) and matches that are genuinely new since the last pass.
  The caller cares most about the new ones.
- **Avoid needless re-processing.** Don't re-inspect files or findings whose state hasn't changed
  and that the caller isn't asking about again. Spend the pass looking for what the prior pass
  didn't surface — related occurrences, newly relevant areas, references not yet inspected.
- **Report a convergence signal.** State plainly whether this pass produced any *materially new*
  findings versus the prior record. This is the single most useful thing a repeat pass can report:
  it lets the calling workflow decide whether to loop again.

Report the signal — don't act on it. This skill does not decide how many passes to run or when an
investigation (let alone any downstream cleanup) is "complete." It reports what changed since the
last pass; the caller owns the loop.

## Reporting back

Report findings compactly enough that the caller can reason without re-reading the repo, but
don't over-format simple answers. For a general search, a table works well:

| File / Area | Target / Identifier | Lines | Occurrences | References | Inspection Status | Notes |
|---|---|---:|---:|---|---|---|

For a reference-tracing investigation, use the structured block format shown in
`references/reference-tracing.md` (definition, occurrences, direct references, what was/wasn't
inspected and why).

Always state plainly:
- What was searched, and where scope was inferred vs. given
- Any exclusions applied
- Ambiguous or noisy matches that need the caller's domain judgment
- What was *not* inspected, and why (usually: "existing evidence was sufficient")
- For a repeated or verification pass: whether it produced any materially new findings versus the
  prior working record (the convergence signal), and which results are new vs. already known

Never take action on the findings (rename, delete, refactor, classify as safe-to-remove, etc.) —
hand the evidence back and let the caller decide.
