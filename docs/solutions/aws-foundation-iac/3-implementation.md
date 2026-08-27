# Implementation

## Abstract

This phase sets out the sequence that brings live resources under Terraform
without changing their behavior: read-only inventory first, then focused
modules and environment roots, remote state and least-privilege pipeline
access, and dependency-ordered import. It also covers pipeline controls, EKS
lifecycle handling, and the rules for sensitive data.

## Implementation sequence

1. Inventory resource identifiers, effective values, dependencies, and existing
   writers through read-only AWS and Kubernetes queries.
2. Define focused Terraform modules and environment roots without changing live
   behavior.
3. Create remote state and least-privilege pipeline access for each root.
4. Import resources in dependency order and refresh state.
5. Review the first plan until every difference is either removed or explicitly
   approved.
6. Apply one bounded lifecycle change and verify its AWS and workload effects.
7. Make the pipeline the normal write path and document emergency
   reconciliation.

## Pipeline controls

The delivery path separates validation, planning, authorization, mutation, and
verification. Formatting and static validation run early. Root-specific tests
check module contracts and critical values. The plan is retained for review,
and high-impact applies require an explicit operator action.

## EKS lifecycle

Cluster and managed-node versions are pinned. Upgrades are separate changes with
compatibility checks, managed add-on review, node rollout observation, and
workload verification. Importing the cluster is not treated as permission to
upgrade it in the same change.

## Sensitive data

State access is restricted and state is treated as sensitive. Workload secret
values remain outside the Terraform source tree. Pipelines use scoped
credentials and do not expose plans or outputs indiscriminately.
