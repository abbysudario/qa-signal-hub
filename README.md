# 🔁 QA Signal Hub

QA Signal Hub is an event-driven workflow that listens to CI signals from [Lighthouse](https://github.com/abbysudario/lighthouse-qa) and routes them into actionable GitHub Issues. It sits between test execution and engineering response.

Lighthouse produces truth. QA Signal Hub produces action.

---

QA Signal Hub does not run tests. It does not decide what to fix. It observes signals coming out of Lighthouse, classifies them by failure type, and creates trackable GitHub Issues so nothing gets lost between a failing CI run and an engineer's attention.

Every decision in this project is intentional. Deduplication prevents issue spam. Failure classification prevents noise. The workflow is exported and versioned so the logic is auditable without logging into n8n.

---

## 🧰 Stack

n8n, GitHub Actions, GitHub Issues API. Receives signals from [Lighthouse](https://github.com/abbysudario/lighthouse-qa) via webhook.

---

## ⚙️ How it works

```
Lighthouse CI run completes
        |
        | webhook POST
        v
n8n receives signal
        |
        |-- REGRESSION  -> deduplicate -> create GitHub Issue if none open
        |-- ENVIRONMENT -> deduplicate -> create GitHub Issue if none open
        |-- FLAKY       -> log only, no issue
        |-- PASS        -> do nothing
```

See [docs/architecture.md](docs/architecture.md) for the full system design.

---

## ▶️ Setup

**1. Clone the repo**
```bash
git clone https://github.com/abbysudario/qa-signal-hub.git
cd qa-signal-hub
cp .env.example .env
# open .env and fill in N8N_WEBHOOK_URL and GITHUB_TOKEN
```

**2. Import the n8n workflow**

Open your n8n instance and import `workflows/lighthouse-signal.json`. The workflow is pre-configured with all nodes. You will need to reconnect your GitHub credentials after import.

**3. Activate the workflow**

Set the webhook node to production mode and activate the workflow. The production webhook URL is what goes into Lighthouse's `N8N_WEBHOOK_URL` secret in GitHub Actions.

**4. Trigger a signal from Lighthouse**

In the Lighthouse repo, go to GitHub Actions -> Lighthouse CI -> Run workflow -> set `signal_mode` to `true`. This runs the deliberate signal fixtures and fires the webhook with real failure data.

---

## 🛡️ Deduplication

Before creating a GitHub Issue, QA Signal Hub searches for an existing open issue with the same title. If one is found, no new issue is created. This prevents issue spam when the same test fails across multiple CI runs before anyone fixes it.

Issue titles are deterministic:
```
[REGRESSION] Signal Generation - wrong product title assertion
[ENVIRONMENT] Signal Generation - unreachable external resource
```

Same test, same title, every time. The deduplication search is reliable because the title never changes.

---

## 🗺️ Roadmap

| Milestone | Focus | Status |
|---|---|---|
| 1 | Repo foundation: structure, architecture, README | ✅ |
| 2 | n8n webhook receiver wired to Lighthouse CI | ✅ |
| 3 | AI failure classification node | 🔜 |
| 4 | GitHub Issue creation with deduplication | 🔜 |
| 5 | Polish: labels, run URL linking, issue templates | 🔜 |