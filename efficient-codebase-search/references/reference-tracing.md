# Reference Tracing and Dependency Discovery

## The tracing flow

```text
Identifier
    ↓
Locate definition
    ↓
Search repository for usages
    ↓
Collect reference locations
    ↓
Group references
    ↓
Inspect only references required by the caller
```

Return (or internally hold) the compact reference set *before* inspecting individual references:

```text
Identifier: ExampleHelper

Definition:
src/example/ExampleHelper.ext:24

Direct references:
src/rules/RuleA.ext:51
src/ui/ScreenA.ext:104
src/service/ServiceB.ext:88

Reference count: 3
```

Only after that should individual references get inspected — and only the ones the caller's
question actually requires.

## Output block for a completed reference investigation

```text
Identifier: ExampleHelper

Definition:
  path/to/ExampleHelper.ext:24

Occurrences:
  8

Direct references:
  path/to/FileA.ext:51
  path/to/FileB.ext:104
  path/to/FileC.ext:88

Inspected:
  FileA.ext
  FileC.ext

Not deeply inspected:
  FileB.ext

Reason:
  Existing evidence was sufficient.
```

Always include the "Not deeply inspected" + "Reason" pair when applicable — it tells the caller
what evidence gap exists, if any, without forcing a full inspection just to be thorough.

## Dependency / fallout discovery

Only run this when the caller asks for fallout or dependency information specifically — it's a
superset of plain reference tracing and costs more.

Look for, as applicable to the language/framework in play:

- Direct references (as above)
- Outgoing dependencies (what the target itself calls/uses)
- Related configuration, UI definitions, metadata
- Related tests
- Inheritance relationships and interface implementations
- Other identifiers likely affected by changing the target

Label **every** fallout finding by confidence — not just the ones that feel surprising or anomalous.
Dependency discovery is inherently less precise than a direct text match, so the label is what tells
the caller how much weight to put on each line:

- **Directly confirmed** — a mechanical search or definition lookup proves the relationship (an
  import, a call, an explicit manifest listing — or an explicit, verified absence, like "no test
  files reference this target").
- **Likely related** — naming, location, or structural pattern suggests a relationship, unconfirmed
  by a mechanical check.
- **Uncertain / requires deeper analysis** — worth flagging, not worth chasing further without
  caller sign-off.

Concretely, this means every line in a "related config" or "related tests" section of the output
should read like "`Foo.xml` — directly confirmed (explicitly lists the target)" or "no matching test
files — directly confirmed absence," not a bare statement of what was or wasn't found. If a fallout
finding doesn't carry one of the three labels, it hasn't gone through this step yet.

Do not recursively traverse the entire dependency graph by default. One hop (direct
references/dependencies) is the default depth; go further only when the caller asks, or the
one-hop evidence doesn't answer their question.
