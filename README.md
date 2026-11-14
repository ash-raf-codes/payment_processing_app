# PayFlow demo payments app

A payments operations demo with 15 independently runnable FastAPI services and a browser frontend. It includes payment authorization, fraud review, FX quotes, duplicate-request protection, ledgering, settlement, disputes, notifications, webhooks, audit records, and reporting. It is designed as the PR Impact Agent's sample application. Service boundaries and calls are documented in [DEPENDENCY_GRAPH.md](DEPENDENCY_GRAPH.md).

## Run the services together

From this directory, start the gateway and all domain services with Docker Compose:

```bash
docker compose up --build
```

Open http://127.0.0.1:8000. Gateway API docs are at http://127.0.0.1:8000/docs. Each service also has its own `/docs` endpoint on internal ports 8101–8114.

Stop all services with `Ctrl+C` or `docker compose down`.

To run a single service during development, install `requirements.txt` and run, for example:

```bash
uvicorn app.main:app --reload --port 8000
uvicorn app.services.payments.main:app --reload --port 8101
```

Set the service URL environment variables (`PAYMENTS_URL`, `CUSTOMERS_URL`, `VENDORS_URL`, `RISK_URL`, `LEDGER_URL`, `NOTIFICATIONS_URL`, `REPORTING_URL`, `FX_URL`, `FRAUD_URL`, `IDEMPOTENCY_URL`, `SETTLEMENT_URL`, `DISPUTES_URL`, `WEBHOOKS_URL`, `AUDIT_URL`) to override service addresses. Compose wires these automatically.

## Demo behavior

- View and filter payment transactions.
- Submit a mock payment with vendor, customer, and card test details; no real payment network is contacted.
- Inspect payment details, capture manually authorized payments, and issue full or partial refunds.
- Screen payments for fraud, quote settlement currency, enforce idempotency, queue payouts, open disputes, and record audit and webhook events.
- The backend validates customer email/phone, order IDs, amounts, card checksum, expiration, CVC shape, vendor, and payment state transitions.
- Card brand, expiration, and last four digits are retained. Full card number and CVC are never stored.
- API endpoints include `/api/dashboard/summary`, `/api/vendors`, `/api/customers`, `/api/payments` (search, filter, pagination), payment detail, capture, and refund.
- Each service owns its API and in-memory demo state; records reset when that service restarts.
- The gateway health endpoint reports service availability at `/api/health`.

All payment details are fictional. Do not enter real card data. This service split demonstrates service boundaries and synchronous HTTP dependencies; it is a hackathon demo, not a production payment platform.

## Test suites for impact analysis

Install the test dependencies with `pip install -r requirements-dev.txt`, then run `pytest -q`. Tests are grouped by service; gateway tests cover the multi-service payment path and graph tests check impact traversal. The current suite has 54 cases. A vendor change fans out to Vendor, Fraud, Risk, Webhooks, and Gateway coverage; a gateway change has no downstream consumers and stays within Gateway and graph tests.

To compare blind full-suite CI with graph-focused selection, run `python scripts/compare_test_runs.py --changed-service vendors`. The script reports test counts and measured runtime for both runs. Try `payments`, `vendors`, or `gateway` to see how different dependency neighborhoods change the selection. This local benchmark demonstrates the graph idea; the PR Impact Agent CLI will provide the integrated selection in CI.

To create the separate Git root required by `impact-agent`, run `python scripts/export_payflow_demo.py /tmp/payflow-demo` from the agent repository root. The exported repo includes this app's test configuration and service graph metadata. Commit the export as a base snapshot, then create a feature commit for the impact-analysis replay.

## Run impact analysis in GitHub Actions

The workflow in `.github/workflows/pr-impact.yml` indexes the PR base snapshot in MongoDB Atlas, runs `impact-agent analyze` against the PR commits, publishes the Markdown explanation in the Actions run summary, and uploads the JSON report as an artifact. It runs for pull requests targeting `main`, indexes `main` after pushes, and supports manual runs from the Actions tab.

Before using it, add `MONGODB_URI` in the repository's **Settings → Secrets and variables → Actions → New repository secret**. The workflow uses the `pr_impact` database and the stable repository ID `payflow-demo`. Keep the Atlas connection string in the secret; do not commit it in this repository.

To trigger it, push a feature branch and open a pull request to `main`. For a manual run, open **Actions → PR impact tests → Run workflow**; the optional `base_ref` and `head_ref` inputs accept branches or commit SHAs. Ordinary repository secrets are unavailable to workflows triggered by fork pull requests, so run the demo from branches in this repository.

The workflow pins the PR Impact Agent to commit `f16149b` from `xIshanSandhux/mongodb-hackathon`. Update `AGENT_REF` in the workflow to a reviewed, immutable commit SHA when a newer agent version is published.
