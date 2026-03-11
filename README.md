# 🔁 QA Signal Hub

QA Signal Hub is the companion repo to [Lighthouse](https://github.com/abbysudario/lighthouse-qa). It is an event-driven signal routing system built with [n8n](https://n8n.io) that listens to CI signals from Lighthouse and routes them into actionable GitHub Issues.

> 💡 This repo does not work in isolation. It is designed exclusively to receive and act on webhook payloads from [Lighthouse](https://github.com/abbysudario/lighthouse-qa). Set up Lighthouse first before configuring QA Signal Hub.

Lighthouse produces truth. QA Signal Hub produces action.

---

QA Signal Hub does not run tests. It does not decide what to fix. It observes signals coming out of Lighthouse, classifies them by failure type, and creates trackable GitHub Issues so nothing gets lost between a failing CI run and an engineer's attention.

Every decision in this project is intentional. Deduplication prevents issue spam. Failure classification prevents noise. Labels communicate severity at a glance. The workflow is exported and versioned so the logic is auditable without logging into n8n.

---

## 🧰 Stack

n8n, GitHub Issues API. Receives enriched webhook payloads from [Lighthouse](https://github.com/abbysudario/lighthouse-qa) via GitHub Actions CI.

---

## ⚙️ How it works

```
Lighthouse CI run completes
        |
        | webhook POST with enriched payload
        v
n8n Webhook node receives signal
        |
        v
Has Classifications? (IF node)
        |
        |-- empty array -> Exit: No Failures (do nothing)
        |
        |-- classifications present
                |
                v
        Split Classifications (splits array into individual items)
                |
                v
        Route By Type (Switch node)
                |
                |-- REGRESSION  -> Search Existing Issues -> Issue Already Exists?
                |-- ENVIRONMENT -> Search Existing Issues -> Issue Already Exists?
                |-- FLAKY       -> Search Existing Issues -> Issue Already Exists?
                |-- UNKNOWN     -> Search Existing Issues -> Issue Already Exists?
                                          |
                                          |-- true  -> Exit: Duplicate (do nothing)
                                          |
                                          |-- false -> Create GitHub Issue with label
```

See [docs/architecture.md](docs/architecture.md) for the full system design.

---

## 📦 Webhook payload

Lighthouse sends the following payload to QA Signal Hub after every CI run:

```json
{
  "status": "failure",
  "signal_mode": "true",
  "branch": "main",
  "commit": "abc123",
  "run_url": "https://github.com/abbysudario/lighthouse-qa/actions/runs/...",
  "classifications": [
    {
      "test": "wrong product title assertion",
      "file": "signal.spec.ts",
      "type": "REGRESSION",
      "confidence": "high",
      "error": "Expected 'This Product Does Not Exist' but received 'Sauce Labs Backpack'"
    },
    {
      "test": "inconsistent inventory item rendering",
      "file": "signal.spec.ts",
      "type": "FLAKY",
      "confidence": "high",
      "error": "Expected 'This Will Fail On First Attempt' but received 'Sauce Labs Backpack'"
    },
    {
      "test": "unreachable external resource",
      "file": "signal.spec.ts",
      "type": "ENVIRONMENT",
      "confidence": "high",
      "error": "Failed to navigate to unreachable host 'https://this-environment-does-not-exist.internal/'"
    }
  ]
}
```

When no failures occur, `classifications` is an empty array. QA Signal Hub always receives the same payload shape regardless of run outcome. An empty array communicates a clean run. This is intentional. Consistent payload shape means the workflow never has to guard against a missing field.

---

## 🏷️ GitHub Issue labels

QA Signal Hub creates GitHub Issues in the [Lighthouse repo](https://github.com/abbysudario/lighthouse-qa) with labels that communicate failure type at a glance. These labels must exist in the Lighthouse repo before the workflow can apply them.

| Label | Color | Description |
|---|---|---|
| `regression` | Red | Test assertion no longer matches application behavior |
| `environment` | Yellow | Infrastructure or external dependency is unreachable |
| `flaky` | Orange | Test passes and fails intermittently across runs |
| `unknown` | Grey | Failure pattern could not be confidently classified |

---

## 🛡️ Deduplication

Before creating a GitHub Issue, QA Signal Hub uses the GitHub Search API to check for an existing open issue with the same title. If one is found, the workflow exits cleanly without creating a duplicate. If none is found, the issue is created.

Issue titles are deterministic:
```
[REGRESSION] wrong product title assertion
[ENVIRONMENT] unreachable external resource
[FLAKY] inconsistent inventory item rendering
```

Same test, same title, every time. The deduplication search is reliable because the title never changes across runs. This prevents issue spam when the same test fails across multiple CI runs before anyone fixes it.

---

## 📋 GitHub Issue format

Each issue created by QA Signal Hub includes the full context needed to investigate and reproduce the failure:

```
**Test:** wrong product title assertion
**File:** signal.spec.ts
**Type:** REGRESSION
**Confidence:** high
**Error:** Expected 'This Product Does Not Exist' but received 'Sauce Labs Backpack'
**Run URL:** https://github.com/abbysudario/lighthouse-qa/actions/runs/...
**Branch:** main
**Commit:** abc123
```

---

## ▶️ Setup

**1. Clone the repo**
```bash
git clone https://github.com/abbysudario/qa-signal-hub.git
cd qa-signal-hub
```

**2. Set up your n8n instance**

QA Signal Hub runs on n8n. You can use [n8n Cloud](https://n8n.io) or self-host. Import `workflows/lighthouse-signal.json` into your n8n instance. The workflow is pre-configured with all nodes and connections.

**3. Create a GitHub fine-grained token**

Go to `https://github.com/settings/tokens?type=beta` and create a token with:
- Repository access: `lighthouse-qa` only
- Permissions: Issues — Read and write
- Expiration: 90 days

This follows the principle of least privilege. The token can only create issues in one repo.

**4. Add the GitHub credential in n8n**

In your n8n workflow, add a GitHub API credential using the fine-grained token. Name it `lighthouse-qa-issues`. This credential is used by both the `Search Existing Issues` and `Create an issue` nodes.

**5. Create labels in the Lighthouse repo**

Go to `https://github.com/abbysudario/lighthouse-qa/labels` and create the four labels listed in the GitHub Issue labels section above.

**6. Add the webhook URL to Lighthouse**

Copy the production webhook URL from your n8n webhook node and add it as a secret named `N8N_WEBHOOK_URL` in the Lighthouse repo at `https://github.com/abbysudario/lighthouse-qa/settings/secrets/actions`.

**7. Activate the workflow**

Set the workflow to active in n8n. The production webhook URL is now live and ready to receive signals.

**8. Trigger a signal from Lighthouse**

In the Lighthouse repo, go to GitHub Actions -> Lighthouse CI -> Run workflow -> set `signal_mode` to `true`. This runs the deliberate signal fixtures and fires the webhook with real failure classifications.

---

## 🗺️ Roadmap

| Milestone | Focus | Status |
|---|---|---|
| 1 | Repo foundation: structure, architecture, README | ✅ |
| 2 | n8n webhook receiver wired to Lighthouse CI | ✅ |
| 3 | Signal routing, deduplication, and GitHub Issue creation | ✅ |
| 4 | Self-hosted n8n via Docker | 🔜 |