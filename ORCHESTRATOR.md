# Understanding orchestration

## Purpose

Orchestration turns a larger engineering request into a controlled sequence of smaller tasks. Each task is assigned to the least expensive worker that can perform it reliably. More capable models are reserved for ambiguity, design decisions, implementation, and final review.

A worker can be a deterministic tool, a local model, a subscription model, or a human approval step. The orchestrator manages dependencies, evidence, budgets, retries, and boundaries.

## Worked use case: issue to reviewed pull request

A realistic request is:

> Take the next approved issue, understand the affected repository, implement the change, run the checks, and prepare a pull request for review.

The work is divided like this:

1. **Intake:** read the issue and confirm its repository, scope, and acceptance criteria.
2. **Reconnaissance:** use inexpensive workers in parallel to map the relevant files, identify existing tests, inspect recent related changes, and list likely risks.
3. **Plan:** give those findings to a stronger model to produce an implementation plan with explicit assumptions.
4. **Implement:** let the stronger coding worker edit an isolated worktree.
5. **Verify:** run deterministic tests and let a lower-cost reviewer check the diff against the acceptance criteria.
6. **Escalate:** send only disagreements, failures, or ambiguous design choices back to the stronger model.
7. **Prepare:** create a pull request draft with the change summary, evidence, and remaining questions. Merging stays behind human approval.

The inexpensive workers handle breadth and repetition. The stronger worker handles judgment. The human sees a focused, reviewable result instead of an unfiltered chain of model activity.

## Flow definition

The flow can be written as YAML:

```yaml
flow: issue-to-reviewed-pr
inputs:
  issue:
    source: github.issue
    read_only: true
  repository:
    source: github.repository
    read_only: true

profiles:
  local-fast:
    provider: local
    purpose: file-inventory-and-test-discovery
  subscription-cheap:
    provider: subscription
    purpose: bounded-reconnaissance-and-diff-review
  subscription-strong:
    provider: subscription
    purpose: planning-implementation-and-ambiguity

steps:
  - id: intake
    type: github.issue.read
    input: issue
    output: requirements

  - id: inspect-files
    type: repository.inspect
    profile: local-fast
    input: repository
    output: file_map
    scope: requirements
    parallel_group: reconnaissance

  - id: inspect-tests
    type: repository.tests
    profile: local-fast
    input: repository
    output: test_map
    scope: requirements
    parallel_group: reconnaissance

  - id: inspect-history
    type: repository.history
    profile: subscription-cheap
    input: repository
    output: related_changes
    scope: requirements
    parallel_group: reconnaissance

  - id: plan
    type: model.plan
    profile: subscription-strong
    input: [requirements, file_map, test_map, related_changes]
    output: implementation_plan
    require_assumptions: true

  - id: implement
    type: repository.edit_worktree
    profile: subscription-strong
    input: implementation_plan
    output: change_set
    approval: required-before-write

  - id: verify
    type: repository.verify
    input: change_set
    output: verification
    commands: [tests, lint, format, secret_scan]

  - id: review
    type: model.review_diff
    profile: subscription-cheap
    input: [requirements, change_set, verification]
    output: review

  - id: prepare-pr
    type: github.pull_request.draft
    input: [requirements, change_set, verification, review]
    output: pull_request
    approval: required-before-external-write

policy:
  max_cost_usd: 2.00
  network: approved-repositories-only
  external_writes: approval-required
  retain_receipt: true
```

The profile field selects the worker lane. The parallel reconnaissance steps are the farming mechanism: several small, bounded jobs produce evidence at low cost. The stronger model receives that evidence instead of repeatedly rediscovering it. Approval fields prevent edits or GitHub writes from becoming invisible side effects.

## Reliability rules

- Retry transient failures within a bounded limit.
- Use a declared fallback only when it is approved for that task.
- Stop when evidence is missing or the result is ambiguous.
- Never silently substitute a more expensive provider.
- Never merge, publish, delete, or modify an external system without approval.

## Run receipts

Every run should record the flow version, issue and repository, steps, providers, model names, duration, estimated cost, files changed, commands run, approvals, and errors. A receipt makes the run auditable and replayable.

## What the files mean

`ORCHESTRATOR.md` explains the pattern. A flow file such as `flows/issue-to-reviewed-pr.yaml` is the machine-readable plan. `AGENTS.md` or provider instructions guide one worker during one step. An executor is the component that loads the flow, routes workers, enforces policy, and records results.

This repository is a communication example, not a finished Arete runtime. The YAML is designed to make the orchestration visible and eventually executable; copying it into an agent's Markdown instructions would describe the plan but would not implement scheduling, routing, receipts, or approvals.

## Scope

The format is intentionally small. It describes routing and policy rather than attempting to become a general-purpose programming language or swarm scheduler.


## Independent review

A review is valuable when it can catch the original worker's blind spots. The safest default for consequential work is to use a different model family or provider for the reviewer, with a separate prompt and an independent context window. This reduces correlated errors.

A different model is not a guarantee. Review quality also depends on a clear acceptance checklist, access to the original requirements, source evidence, and deterministic tests. For low-risk work, the same model family can be acceptable when the reviewer is given a pessimistic rubric and cannot simply repeat the original answer. For security-sensitive, production, financial, or externally published work, use a genuinely independent model or human review and require agreement before proceeding.

In a flow, this can be expressed as:

```yaml
- id: review
  type: model.review_diff
  profile: subscription-cheap-independent
  independence:
    different_provider: preferred
    separate_context: required
    rubric: acceptance-criteria-and-failure-modes
```

The rule belongs in the orchestration policy because it governs relationships between workers. Individual `AGENTS.md` files can describe how to review, but they should not be the only place that independence is enforced.
