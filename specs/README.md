# Spec format

File name: `S<NN>-<module>.md` (module numbers: see specs/PARITY.md).
Plain, simple language. Behavior, not screens.

```
# S<NN> <Module>

## Scope
What this module covers and what it leaves to other modules (link them).

## Entities
Table: entity | field | type | required | notes

## Rules
Numbered (R1, R2…). Each rule is testable. Tag where it came from:
[src:N] = from source N, [decision] = Wareslate decision (public sources silent or unclear).

## User actions
Who (role) can do what. Link to S01 for roles.

## Acceptance tests
Given / When / Then, each referencing rule ids.

## Tier
Suggested packaging tier, or "unknown".

## Open questions
Things to decide later.

## Sources
Provenance for [src:N] tags is kept in private research notes.
```
