# Agentic AIOps with Amazon Bedrock

> **Phase:** 5 Operate · **Category:** Modernization
> **Priority topic:** 32
> **Evidence standard:** The retained implementation supports issue intake,
> operator-started analysis, tool-assisted evidence gathering, reports, and
> approval-gated remediation. It does not support a claim of autonomous
> production remediation or a verified percentage reduction in resolution time.

## Abstract

Agentic AIOps augments an evidence-based operating model. It does not replace
service ownership, observability, runbooks, access control, or incident command.

## Control boundary

```text
alert or operator-created issue
  -> immutable issue and scope record
    -> operator starts analysis
      -> read-only tools gather bounded evidence
        -> model proposes diagnosis, confidence, and alternatives
          -> deterministic policy classifies requested action risk
            -> authorized human approves or rejects
              -> narrow automation executes and verifies
                -> audit, result, and rollback record
```

## Design requirements

| Concern | Required control |
|---|---|
| Prompt and evidence trust | Treat logs, tickets, resource names, and retrieved text as untrusted data; isolate instructions from evidence |
| Identity | Separate human, agent, analysis-tool, and remediation identities |
| Authorization | Allowlist tools, targets, verbs, namespaces/accounts, parameter shapes, and execution duration |
| Approval | Use deterministic risk tiers and meaningful human review for consequential actions |
| Privacy | Redact credentials and sensitive payloads before model access; define retention and Region requirements |
| Evaluation | Test correctness, tool selection, abstention, safety, latency, cost, and failure behavior |
| Audit | Record issue, evidence references, model/version, tool calls, reviewer, decision, action, verification and rollback |

## Implementation sequence

1. Begin with read-only summarization and evidence correlation for a bounded
   incident class.
2. Build a retained evaluation set with expected diagnosis, missing evidence,
   unsafe actions, and required abstention.
3. Add tool calls using least-privilege, time-bound identities and parameter
   validation outside the model.
4. Compare recommendations with operator decisions and measure false confidence,
   omitted evidence, cost, and time.
5. Introduce remediation only for narrow, reversible actions with approval,
   preconditions, postconditions, timeout, and rollback.
6. Revoke automation when evaluation or runtime safety regresses.

## Acceptance evidence

- the system refuses out-of-scope or insufficient-evidence requests;
- untrusted log text cannot expand tool authorization;
- high-risk actions cannot execute without the required reviewer;
- every action is attributable and independently reproducible from evidence;
- failed verification stops expansion and triggers rollback or escalation;
- operators can disable the agent without disabling core monitoring or runbooks.

## References

- [Agentic AI Lens: human-in-the-loop for critical decisions](https://docs.aws.amazon.com/wellarchitected/latest/agentic-ai-lens/agentsec04-bp02.html)
- [Agentic AIOps on Amazon Bedrock and Amazon EKS](../agentic-aiops-bedrock/README.md)
