# RiskLab

RiskLab is a learning and portfolio project for building an explainable credit-decision service in Python. It evaluates synthetic loan applications with explicit, versioned rules, saves the baseline result, and compares candidate policies without changing the original decision.

All applicants, policy rules, and outcomes in this repository are invented for the exercise.

## Scope

The first version deliberately covers:

- synthetic applicants;
- one synthetic loan product;
- one currency, supplied through the fixed `DEMO_CURRENCY` project setting;
- deterministic rule-based decisions rather than machine learning;
- three possible outcomes: `APPROVE`, `REVIEW`, and `DECLINE`.

`DEMO_CURRENCY` labels the monetary values used in a run. Changing that label does not change the exercise rules. Currency conversion and exchange rates are outside the project.

`REVIEW` means that the synthetic application would require human attention. A reviewer interface and an operational review workflow are outside the project.

The initial scope excludes a frontend, real customer data, real financial integrations, multiple loan products, machine-learning models, Kafka, Kubernetes, Helm, and public deployment.

## Decisions and policy comparison

- `APPROVE` means no decline or review rule matched.
- `REVIEW` means at least one review rule matched and no decline rule matched.
- `DECLINE` means at least one decline rule matched.

The API returns the baseline policy's saved decision. Background processing records what a candidate policy would have decided from the same saved feature snapshot. Synthetic replay evaluates baseline and candidate policies against the same reproducible synthetic dataset and reports their differences. Candidate evaluation never replaces or updates the saved baseline decision.

The exact baseline rules are specified in [docs/policy.md](docs/policy.md). Accepted application data and request outcomes are specified in [docs/contracts.md](docs/contracts.md).

## Success criteria

RiskLab is complete when a reviewer can follow documented commands to:

1. submit a valid synthetic application and receive its saved baseline decision;
2. retrieve that decision and its explanation;
3. inspect the candidate-policy comparison created from the saved feature snapshot;
4. run synthetic replay and identify decision and reason-code differences;
5. demonstrate that duplicate processing is safe and that pending events recover after a broker outage;
6. run the automated unit, contract, integration, type, and CI checks successfully.

The final demonstration will show one valid submission, its saved baseline result and candidate comparison, an invalid request, an identical repeated request, duplicate event delivery, broker-outage recovery through the outbox, policy replay, and passing automated checks.

## Current status

Phase 0 is complete: project scope, policy `v1`, and the initial external contract are documented. Application code has not been implemented yet.
