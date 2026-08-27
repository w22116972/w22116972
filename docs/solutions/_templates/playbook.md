# <Situation Name> Playbook

> **Status:** Draft | Exercised | Verified in incident
> **Last exercised:** <YYYY-MM-DD per path, or "Never">
> **Owner:** <Role accountable for keeping this playbook correct>

## Abstract

<The class of situation this playbook covers and the decision it helps make. A
playbook classifies first and then commits to one path; if the order of work is
fixed regardless of conditions, write a runbook instead.>

## Roles

| Role | Responsibility |
|---|---|
| <Role> | <What this role decides or performs> |

## Decision frame

<Classify before acting. Each observed condition must select exactly one path.
If two rows can be true at once, state the tie-break rule beneath the table.>

| Observed condition | Path | Confidence required |
|---|---|---|
| <Signal or classification> | <Path A> | <What evidence is enough to commit> |
| <Signal or classification> | <Path B> | <What evidence is enough to commit> |

<If no row matches, stop and escalate. Do not default to the most familiar
path.>

## Paths

### Path A: <name>

**When to use:** <The condition from the decision frame that selects this path,
plus any precondition specific to it.>

**Status:** <Draft | Exercised | Verified in incident> · **Last exercised:**
<YYYY-MM-DD, or "Never">

**Steps:**

1. <Action.>
2. <Action.>

**Exit condition:**

| Item | Pass criteria | Evidence | Owner |
|---|---|---|---|
| <Condition> | <Binary test> | <Artifact> | <Role> |

### Path B: <name>

**When to use:** <Selecting condition.>

**Status:** <Draft | Exercised | Verified in incident> · **Last exercised:**
<YYYY-MM-DD, or "Never">

**Steps:**

1. <Action.>
2. <Action.>

**Exit condition:**

| Item | Pass criteria | Evidence | Owner |
|---|---|---|---|
| <Condition> | <Binary test> | <Artifact> | <Role> |

## Abort conditions

| Condition | Why it is disqualifying | Action |
|---|---|---|
| <Observation> | <Risk it signals> | <Abort, escalate, or switch path> |

<Switching paths mid-execution requires re-entering the decision frame and
recording why the original classification was wrong.>

## Evidence record

| Field | Value |
|---|---|
| Operator and approver | |
| Start and end time | |
| Observed condition and classification | |
| Path selected | |
| Path changes and reason | |
| Exit condition results | |
| Outcome | |

## Compliance

<How this playbook is kept true. Paths mature independently, so each path
carries its own status; an unexercised path must not be presented as a
capability.>

| Check | Mechanism | Cadence |
|---|---|---|
| Each path still works | <Drill that exercises that path> | <How often, per path> |
| The decision frame still discriminates | <Review of misclassifications and escalations> | After each run |
| No path is silently stale | <Status and last-exercised review> | <How often> |

## Notes

<Optional. Revision history, paths that have never been exercised, and links to
the incidents that shaped the decision frame.>

## References

- [<Source title>](<url>)
