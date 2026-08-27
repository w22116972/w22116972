# Solution Documentation Conventions

## Abstract

This document is the structural contract for everything under
`docs/solutions/`. It exists so that a reader can predict where information
lives before opening a file, and so that two authors describing the same
concept choose the same section name.

The governing assumption is that a delivery consultant works from these
documents without other assistance. Every phase index must therefore state
what to collect, what proves it, what commonly goes wrong, and what must be
true before the next phase starts.

This contract governs structure and vocabulary. It does not govern the
technical position of any individual solution.

## Phase model

Each solution follows one delivery spine. The phase number is the leading
digit of every filename.

| Phase | Name | Question it answers |
|---|---|---|
| 0 | Context | What is the reference scenario, and what does this solution promise? |
| 1 | Assess | What exists today, and what would success mean? |
| 2 | Design | What is the target, and which decisions produced it? |
| 3 | Execute | How is it built and migrated? |
| 4 | Validate | What proves it works? |
| 5 | Operate | Who runs it, and when is the source retired? |

Phase 0 is optional. Use it when a solution needs an explicit reference
scenario or claim boundary separate from its assessment. Phases 1 through 5
are required for every solution.

There is no optimization phase. Continuous optimization is a separate
engagement and belongs in its own solution directory.

Slot `6` is reserved for an appendix and is not a delivery phase. Today it
holds the interview drill. A README lists slot 6 under `Appendix`, never inside
its `Phases` list, so that the delivery spine stays five phases long.

## Depth model

Depth is expressed by the number of numeric segments in the filename.

| Pattern | Role |
|---|---|
| `README.md` | Solution narrative, claim boundary, and phase index |
| `N-*.md` | Phase main: the phase's content, its exit criteria, and its index when it has children |
| `N-M-*.md` | A single domain within phase N |
| `N-M-K-*.md` | A narrower topic within that domain |

A phase main is required for every phase. Most phase mains carry the phase's
content directly; only a phase that has `N-M-*` children also acts as an index
for them. A child document must never occupy its parent's slot in the README
phase list.

## Document types

Type is orthogonal to phase and depth. Three dimensions, one carrier each:

| Dimension | Carrier |
|---|---|
| Phase | Leading number, `N-` |
| Depth | Count of numeric segments |
| Type | Filename suffix |

| Type | Filename | Character |
|---|---|---|
| Solution | `N-description.md` | Narrative: what, why, and which tradeoffs |
| ADR | `N-M-description-adr.md`, or a repeated block inside `Key decisions` | One decision, its rejected alternatives, and how adherence is checked |
| Interview drill | `6-description-interview.md` | One resume claim, examined question by question in STAR form |
| Runbook | `N-M-description-runbook.md` | Ordered, single operator, every step verifiable |
| Playbook | `N-M-description-playbook.md` | Branching: classify first, then follow one path |
| Script | `scripts/` with a real extension | Executable commands |

Solution is the default and carries no suffix. Because the suffix names the
type, no in-document type badge is required.

A runbook or playbook may sit at any depth below the phase index. Type is not
a function of depth.

Checklist is not a type. Verification belongs inside the document that
performs the work, as a gate table (see below).

## Document header block

Every document opens with an H1 title, then an optional blockquote metadata
block, then `## Abstract`. `Abstract` is always the first section, with one
exception: an interview drill opens with `Resume claim`, because the claim
under examination is what the document is about.

A `README.md` carries no metadata block. It opens with its H1 title and then
`## Abstract`, because a solution index states its claim boundary in prose
rather than as a field.

```markdown
# Workload Identity and Secrets Modernization

> **Phase:** 2 Design · **Category:** Modernization
> **Evidence standard:** The retained solution supports per-workload identity
> and external Secret references. EKS Pod Identity adoption remains a target
> decision unless runtime evidence is retained for a specific workload.

## Abstract
```

Metadata never appears as a section heading. Prose never appears in the
metadata block.

## README narrative shape

A `README.md` is the interviewer-facing document. Its required sections are
ordered so that they map onto the STAR narrative used in behavioral interviews,
without borrowing STAR's section names:

| STAR beat | README section |
|---|---|
| Situation | `Abstract`, first paragraph: the condition and why it mattered |
| Task | `Abstract`, second paragraph: what this solution had to achieve, and its boundary |
| Action | `Architecture` and `Key decisions`: what was built, and which alternatives lost |
| Result | `Results and claim boundary`: what was measured, and what is not claimed |

Do not rename these sections to `Situation`, `Task`, `Action`, and `Result`.
STAR is a framework for compressing one story into a spoken answer; a README is
also a reference document that readers navigate by topic. The mapping gives the
narrative order without costing the document its index value.

`Key decisions` carries most of the interview weight, because the predictable
question is not what was built but why it was chosen over the alternative. Every
ADR under it therefore names its rejected alternatives explicitly.

## Criteria table

Every set of criteria uses one table shape, whatever the document type:

| Item | Pass criteria | Evidence | Owner |
|---|---|---|---|
| Statement of what must be true | Objective, binary test | Artifact that records the result | Accountable role |

It is used wherever conditions are judged: the `Entry criteria` and `Exit
criteria` of a phase index, the `Verify` line of a runbook step, and the `Exit
condition` of a playbook path.

## Canonical section names

Use the canonical name. Do not introduce a synonym for a concept that already
has one.

| Canonical | Use for | Do not use |
|---|---|---|
| `Abstract` | Opening summary, the first section of every document | `Executive summary`, `Purpose`, `Role in the journey`, `Delivery principle` |
| `Scope` | What the work covers and excludes, when it needs its own section | — |
| `Phases` | A README's ordered index of its phase documents, numbered from 0 | `Lifecycle`, `Detailed design`, `Detailed implementation` |
| `Appendix` | A README's list of slot-6 documents, kept out of `Phases` | — |
| `Resume claim` | The verbatim resume bullet an interview drill defends | — |
| `Evidence boundary` | Claim-by-claim mapping to what the source proves | — |
| `Problem statement` | The condition the work addresses | `Customer problem`, `Customer context`, `Business problem` |
| `Evidence standard` | Claim boundary and evidence grading | `Evidence status`, `Scope and evidence standard` |
| `Architecture` | Target structure | `Target pattern`, `Recovery architecture` |
| `Key decisions` | A section holding one or more ADR blocks | `Architecture decisions`, `Key decisions and tradeoffs`, `Decision table` |
| `Context` | The forces that made a decision necessary | `Reason`, `Background` |
| `Rejected alternatives` | Options considered and why each lost | `Rejected alternative`, `Alternatives considered` |
| `Tradeoff` | What a decision buys and what it costs, ending in its reversal trigger | `Consequences`, `Tradeoffs` |
| `Compliance` | How adherence to a decision or procedure is verified | `Evidence required` |
| `Decision frame` | Playbook classification that selects a path | — |
| `Implementation sequence` | Ordered build or migration steps | `Adoption sequence`, `Migration sequence` |
| `Success criteria` | Objectives and how they are judged | `Acceptance criteria` |
| `Acceptance evidence` | What was actually retained as proof | `Verification performed` |
| `Entry criteria` | Conditions required before starting | `Preconditions`, `Prerequisites` |
| `Exit criteria` | Conditions required before proceeding | `Exit gate`, `Production gates`, `Handoff gate` |
| `Known pitfalls` | Recurring failure modes and how to avoid them | `Watch-outs` |
| `Rollback` | Reversal path and its boundary | `Rollback boundary` |
| `Roles` | Who supplies input and who signs off, for this document | — |
| `Ownership model` | Durable ownership of platform layers, distinct from `Roles` | — |
| `Evidence record` | What an operator must record while executing | — |
| `References` | External sources | `Related guidance` |

`Problem statement` describes a condition, not a party at fault. Write it
without attributing the condition to the customer.

Section names not listed here are free, provided they do not restate a
canonical concept under a different label.

## Required sections

### Solution, phase main (`N-*.md`)

Always required:

```
Abstract · [content sections drawn from the canonical vocabulary]
· Exit criteria · References
```

Additionally required when the phase has `N-M-*` children:

```
Entry criteria · Roles · Domain index · Known pitfalls · Continue
```

`Domain index` is a table of the domains this phase covers, what each must
produce, which document holds the detail, and who owns it. `Continue` links
to the next phase and to this phase's child documents.

A phase main without children does not need those five sections. Do not turn a
content document into a thin index to satisfy a template.

### Solution, phase content (`N-M-*.md`, `N-M-K-*.md`)

```
Abstract · [content sections drawn from the canonical vocabulary]
· Acceptance evidence · References
```

### ADR

```
Context · Decision · Rejected alternatives · Tradeoff · Compliance · Notes
```

An ADR carries `Status`, `Date`, and `Owner` in its metadata block. `Status` is
one of Proposed, Accepted, Superseded by ADR n, or Deprecated. A decision made
before this repository existed uses `Date: Not retained` and says how the
record was reconstructed; inventing a date is worse than admitting the gap.

`Rejected alternatives` is required. An ADR with no rejected alternative is a
note, not a decision record. `Tradeoff` states both directions and ends with
the reversal trigger that would require a new ADR. `Compliance` names the test,
gate, policy, or review that would catch a violation, because a decision no one
checks is a preference.

An ADR may be its own `*-adr.md` file, or a repeated `### ADR n: <title>` block
inside a `Key decisions` section. The field set is the same either way; as a
block, the metadata fields appear as bold labels rather than a blockquote.

### Interview drill (`6-*-interview.md`)

```
Resume claim · Evidence boundary · <numbered questions>
· Related implementation phases
```

`Resume claim` quotes the resume bullet verbatim as a blockquote, so the reader
sees exactly what is being defended. `Evidence boundary` maps each element of
that claim to what the current source actually proves and to an interview-safe
way to state it; a claim the checkout cannot support is corrected here rather
than repeated.

Each numbered question is a `##` heading phrased as an interviewer would ask
it, containing `### Situation`, `### Task`, `### Action`, and `### Result`.
Every question stands alone, so an interviewer can drill into one decision
without requiring the whole project narrative.

STAR belongs to this type only. Do not restructure phase documents into
Situation, Task, Action, and Result: STAR compresses one story into a spoken
answer, while a phase document is also a reference that readers navigate by
topic.

### Runbook (`*-runbook.md`)

```
Abstract · Roles · Entry criteria · Procedure · Abort conditions · Rollback
· Evidence record · Compliance · Notes · References
```

Every numbered step in `Procedure` carries its own verification. A step
without a stated way to confirm it succeeded is not a runbook step.

A runbook carries `Status`, `Last exercised`, and `Owner` in its metadata
block. `Compliance` states how the procedure is kept true, because an
unexercised runbook decays silently while still looking authoritative.

### Playbook (`*-playbook.md`)

```
Abstract · Roles · Decision frame · Paths · Abort conditions
· Evidence record · Compliance · Notes · References
```

`Decision frame` maps an observed condition to exactly one path. Each entry
under `Paths` states when to use it, its steps, and its exit condition, and
carries its own `Status` and `Last exercised`, because paths mature
independently. An unexercised path must not be presented as a capability.

### Script

```
Abstract · Entry criteria · Parameters · Usage · Expected output
· Failure handling · Safety notes
```

## Numbering rules

- Numbers are stable identifiers, not a contiguous sequence.
- Gaps are permitted and carry no meaning; do not renumber to close them.
- A retired number is never reused for different content.
- Adding a document must not require renumbering an existing one.

## Evidence language

Solutions in this repository separate what is proven from what is proposed.
Use these three labels consistently and do not blur them:

| Label | Meaning |
|---|---|
| Verified | Supported by retained source, inventory, or artifact |
| Historical claim | Recorded but lacking the baseline or calculation for replay |
| Acceptance target | A gate a real engagement must approve and measure |

A shared document used by more than one solution must be written entirely in
acceptance-target voice. It cannot inherit one solution's verified findings,
because doing so would extend that claim to solutions with no such evidence.

## Formatting

- Wrap prose at 80 characters. Tables, headings, links, and code blocks are
  exempt.
- Use ATX headings and sentence case, preserving product capitalization.
- No trailing whitespace.

See [Markdown Style Guide](../practices/style/markdown-style-guide.md) for the
full rules.

## Templates

- [Phase main](_templates/solution-phase-main.md)
- [ADR](_templates/adr.md)
- [Phase content](_templates/solution-phase-content.md)
- [Runbook](_templates/runbook.md)
- [Playbook](_templates/playbook.md)
