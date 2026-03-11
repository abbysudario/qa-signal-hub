# QA Signal Hub — Architecture

## Overview

QA Signal Hub is an event-driven workflow that listens to CI signals from Lighthouse and routes them into actionable GitHub Issues. It sits between test execution and engineering response — it does not run tests, it does not decide what to fix. It observes signals, classifies them, and creates trackable records so nothing gets lost.

Lighthouse produces truth. QA Signal Hub produces action.

---

## Signal flow

```
Lighthouse CI run completes
        |
        | webhook POST (status, signal_mode, branch, commit, run_url)
        v
n8n webhook receiver
        |
        v
Classify failure type
        |
        |-- REGRESSION  --> deduplicate --> create GitHub Issue if none open
        |-- ENVIRONMENT --> deduplicate --> create GitHub Issue if none open
        |-- FLAKY       --> log only, no issue (needs trend analysis first)
        |-- PASS        --> do nothing
```

---

## Components

**Lighthouse** triggers the webhook at the end of every CI run. The payload includes the job status, signal mode flag, branch, commit SHA, and a direct link to the CI run. When `signal_mode=true`, the run includes deliberate signal fixtures that produce REGRESSION, FLAKY, and ENVIRONMENT failures.

**n8n** is the workflow engine. It receives the webhook, evaluates the payload, and executes the routing logic. Each node in the workflow does one thing. The visual editor makes the logic auditable without reading code.

**GitHub Issues** is the output layer. Issues are the canonical record of a known failure. They are searchable, assignable, and closeable. When a signal produces an issue, the run URL is linked so engineers can jump directly to the CI failure.

---

## Deduplication

Before creating a GitHub Issue, QA Signal Hub searches for an existing open issue with the same title. If one exists, no new issue is created. This prevents issue spam when the same test fails across multiple CI runs before anyone has time to fix it.

Issue titles are deterministic so the deduplication search is reliable:

```
[REGRESSION] Signal Generation - wrong product title assertion
[ENVIRONMENT] Signal Generation - unreachable external resource
```

Same test, same title, every time.

---

## Failure classification

| Signal | Behavior | Rationale |
|---|---|---|
| REGRESSION | Create GitHub Issue | A test assertion no longer matches the app. Something broke and needs investigation. |
| ENVIRONMENT | Create GitHub Issue | A dependency is unreachable. Infrastructure needs attention. |
| FLAKY | Log only | Flaky tests need trend data before escalation. One flaky run is noise. A pattern is a signal. |
| PASS | Do nothing | No action needed. |

---

## Repositories

- **Lighthouse:** https://github.com/abbysudario/lighthouse-qa
- **QA Signal Hub:** https://github.com/abbysudario/qa-signal-hub