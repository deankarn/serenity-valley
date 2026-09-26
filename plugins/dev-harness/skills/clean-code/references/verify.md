# Verify

Pick the checklist by what is in context. A proposed plan or interface design → **Plan audit**. Source files, a diff, or a repo → **Code audit**. Both present → run both, plan first.

State which one you ran. Report findings grouped by file or interface, each with a severity and the specific location. Explicitly list what is already correct and should be left alone.

Severity:
- **convention** — breaks one of the conventions. Fix before merging.
- **note** — smell, judgement call, or something to watch.

Give each finding a one-line fix direction. Do not rewrite code during an audit unless asked. Report, then offer.

---

## Plan audit

1. Are data lookups named `get*` (zero or one, optional) and `find*` (zero or more, collection)?
2. Is absence modelled as an empty optional or collection, never as an error?

---

## Code audit

Use Grep and Glob. Adapt the patterns to the language — these are signals, not literal searches.

**Lookup naming**
- `get*` returning a collection, or throwing or erroring when nothing is found.
- `find*` returning a single item, a nullable, or nil.
- Lookups returning a bare pointer or null for absence where the project has an Option type.
- Lookups whose doc comment does not state the return shape.

**Variable names**
- Terse names (one or two letters, cryptic abbreviations) whose scope spans more than about 40 lines. Loop counters are exempt.
- Names longer than the concept needs.

**Formatting**
- Reformatting of untouched code mixed into a logic change. Which formatter and settings the project uses is its decision; do not audit them.
