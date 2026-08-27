# <Phase Name for This Solution>

> **Phase:** <N> <Context|Assess|Design|Execute|Validate|Operate>
> **Evidence standard:** <What this phase can prove, and what remains an
> acceptance target.>

## Abstract

<What this phase covers for this solution, in two or three sentences. State
the boundary: what this phase decides, and what it deliberately defers to a
later phase or a separate solution.>

## Entry criteria

<Required only when this phase has `N-M-*` child documents.>

<What must already be true before this phase starts. Reference the previous
phase's exit criteria rather than restating them.>

| Item | Pass criteria | Evidence | Owner |
|---|---|---|---|
| <Condition> | <Binary test> | <Artifact> | <Role> |

## Roles

<Required only when this phase has `N-M-*` child documents.>

| Role | Supplies | Signs off |
|---|---|---|
| <Role> | <Input this role must provide> | <Decision this role approves> |

One person may hold several roles, but every decision is still attributed to
a named role.

## Domain index

<Required only when this phase has `N-M-*` child documents.>

| Domain | Required output | Detail | Owner |
|---|---|---|---|
| <Domain> | <What this phase must produce for it> | [<Document>](<N-M-file.md>) | <Role> |

<Domains without a linked document are covered inline in this file. If a
domain grows past a section, promote it to an `N-M-*` document and link it
here.>

## Known pitfalls

<Required only when this phase has `N-M-*` child documents.>

| Pitfall | Why it happens | How to avoid it |
|---|---|---|
| <Recurring failure mode> | <Root cause> | <Control or test> |

## Exit criteria

<This phase is complete only when every row passes. An unknown is acceptable
only with a named owner, a decision date, and a safe fallback.>

| Item | Pass criteria | Evidence | Owner |
|---|---|---|---|
| <Condition> | <Binary test> | <Artifact> | <Role> |

## Continue

<Required only when this phase has `N-M-*` child documents.>

- Next phase: [<Phase N+1 name>](<file.md>)
- Domain detail: [<Document>](<N-M-file.md>)
- Procedures: [<Runbook or playbook>](<N-M-file-runbook.md>)

## References

- [<Source title>](<url>)
