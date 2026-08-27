# <Procedure Name> Runbook

> **Status:** Draft | Exercised | Verified in incident
> **Last exercised:** <YYYY-MM-DD, or "Never">
> **Owner:** <Role accountable for keeping this procedure correct>

## Abstract

<What this procedure accomplishes, when it is invoked, and what it must not be
used for. State the one condition that means this is the wrong runbook.>

<Where a command would expose environment identifiers, stay provider-neutral
and describe the action instead.>

## Roles

| Role | Responsibility |
|---|---|
| <Role> | <What this role decides or performs> |

One person may fill several roles in a small exercise, but each decision must
still be attributed.

## Entry criteria

<Everything that must be true before step 1. If any row cannot be confirmed,
stop and escalate rather than proceeding on assumption.>

| Item | Pass criteria | Evidence | Owner |
|---|---|---|---|
| <Condition> | <Binary test> | <Artifact> | <Role> |

## Procedure

<Each step states the action, then how to confirm it worked. A step with no
verification is not a step.>

1. <Action.>
   - **Verify:** <Objective, observable confirmation.>
2. <Action.>
   - **Verify:** <Objective, observable confirmation.>

<Where the procedure branches on a condition known before execution, use a
playbook instead. Where it branches on a result discovered during execution,
add a labelled path below and return to the numbered sequence.>

## Abort conditions

<Stop immediately and follow `Rollback` if any of these occur. Do not
improvise past an abort condition.>

| Condition | Why it is disqualifying | Action |
|---|---|---|
| <Observation> | <Risk it signals> | <Abort, escalate, or roll back> |

## Rollback

<How to return to the prior state, and the point beyond which return is no
longer possible. Name the last reversible point explicitly.>

1. <Reversal step.>
   - **Verify:** <Confirmation that the prior state is restored.>

## Evidence record

<Record these while executing, not afterwards.>

| Field | Value |
|---|---|
| Operator and approver | |
| Start and end time | |
| Entry criteria confirmed | |
| Steps completed | |
| Deviations and exceptions | |
| Verification results | |
| Outcome | |

## Compliance

<How this runbook is kept true. A procedure that is never exercised decays
silently, and its `Status` above must reflect that.>

| Check | Mechanism | Cadence |
|---|---|---|
| The procedure still works | <Drill or exercise that proves it> | <How often> |
| The procedure still matches the system | <Change events that force a re-exercise> | On change |
| Deviations feed back | <How exceptions become runbook edits> | After each run |

## Notes

<Optional. Revision history, known gaps, and links to the incidents or drills
that shaped this procedure.>

## References

- [<Source title>](<url>)
