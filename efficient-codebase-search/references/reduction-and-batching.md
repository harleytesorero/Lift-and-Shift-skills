# Result Reduction and Batching

## Reduction example

Raw mechanical search output:

```text
Foo.gs:42 Example
Foo.gs:51 Example
Foo.gs:70 Example
Foo.gs:81 Example
```

Reduced to a compact record:

```text
File: Foo.gs
Target: Example
Occurrences: 4
Lines: 42, 51, 70, 81
```

Do this per file/identifier pair before moving on to inspection. The reduced form is what should
occupy context going forward — not the raw hit list.

## Batching large result sets

When a search returns hundreds of raw matches, don't reason over them one at a time. Collapse in
stages, narrowing at each step:

```text
800 raw matches
      ↓ group by file
120 unique files
      ↓ apply allowed exclusions (build/, generated/, vendor/)
75 relevant search locations
      ↓ group by identifier
25 candidates requiring context
      ↓ selective inspection (Step 4 of the main workflow)
```

Each arrow is a mechanical or bookkeeping operation (dedup, grouping, filtering) — not a reasoning
step. Reasoning only enters once you're down to the small set of candidates that actually need a
closer look.

## Flagging ambiguity without resolving it

Short or overloaded identifiers can have several unrelated meanings across a codebase. Report the
raw shape of the ambiguity and let the caller resolve it:

```text
Target: CIG
Occurrences: 126
Files: 34
Potentially ambiguous short identifier
```

Don't guess which occurrences are "the real ones" — that's a domain judgment call that belongs to
the caller.
