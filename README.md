# Project Sentinel

**When a release goes bad, Sentinel catches it in the canary, rolls it back and collects the evidence. A GitLab Duo flow then proposes a fix together with a regression test. The fix merges only after deterministic gates prove that it works.**

Core rule: **AI proposes; deterministic gates decide.** The Duo flow can read evidence and open a merge request. It cannot merge, deploy, run tests or mark an incident as recovered. Each of those steps is done by a CI job, and every job's result is signed and recorded.

This is our entry for the GitLab Transcend hackathon (Life After Code). It is MIT licensed.

## The loop

```
push ─▶ lint / tests / security ─▶ canary evaluate ─FAIL─▶ rollback ─▶ evidence:collect
                                        │                                    │
                                      PASS                       "pipeline failed" event
                                        ▼                                    ▼
                                     promote               Duo flow: investigator ─▶ repairer
                                                                             │
                                                 MR: fix + regression test + analysis.json
                                                                             ▼
                                   gate:regression-proof  (new test FAILS on broken, PASSES on fix)
                                   gate:integrity-gate    (no weakened tests, no skips, no CI edits)
                                                                             ▼
                                         merge ─▶ pipeline on merge commit ─▶ canary PASS ─▶ RECOVERED
```

Each step sends an HMAC-signed event to the Sentinel backend. The backend runs an evidence-gated state machine:

`DETECTED → ROLLED_BACK → EVIDENCE_COLLECTED → INVESTIGATING → REPAIR_PROPOSED → VERIFYING → MERGED → REDEPLOYING → RECOVERED`

The backend refuses any transition that arrives without its required evidence, and logs the refusal. For example, if an AI message claims the incident is "recovered", that claim is refused.

## Evidence: a real run on this project

| Step | Link |
|---|---|
| A healthy release, with every job green | [pipeline 2992617972](https://gitlab.com/gitlab-ai-hackathon/transcend-october-2026/43057900/showcase/-/pipelines/2992617972) |
| A faulty commit (`refactor(orders): simplify tier discount lookup`). The canary fails (16.67 % errors on the candidate, 0 % on stable), and the evidence is collected | [pipeline 2992630194](https://gitlab.com/gitlab-ai-hackathon/transcend-october-2026/43057900/showcase/-/pipelines/2992630194) |
| The failed pipeline triggers the Duo flow, which opens the repair MR | [MR !1](https://gitlab.com/gitlab-ai-hackathon/transcend-october-2026/43057900/showcase/-/merge_requests/1) |
| On the MR, the regression proof is **PROVEN** and the Integrity Gate gives **PASS** | [pipeline 2992662315](https://gitlab.com/gitlab-ai-hackathon/transcend-october-2026/43057900/showcase/-/pipelines/2992662315) |
| After the merge, the merge commit passes the canary: 0.00 % errors on the candidate, against 16.67 % on the old broken release | [pipeline 2992674535](https://gitlab.com/gitlab-ai-hackathon/transcend-october-2026/43057900/showcase/-/pipelines/2992674535) |

## What is in the repo

| Path | Contents |
|---|---|
| `app/` | The demo service, a FastAPI orders API, which Sentinel releases, breaks and repairs |
| `backend/` | The Sentinel backend (FastAPI + SQLite): signed event intake, the incident state machine, GitLab webhooks and the audit ledger |
| `frontend/` | The Mission Control dashboard (React + Vite), with pages for rollout, incidents, gates, the ledger and system status |
| `sentinel/scripts/` | The pipeline scripts: `evaluator.py`, `canary_deploy.py`, `rollback.py`, `evidence_collector.py`, `regression_proof.py`, `integrity_gate.py`, `report_gate_verdict.py` and `auto_merge.py`, plus the security, FinOps and governance gates |
| `.gitlab/duo/flows/sentinel-investigate-repair.yml` | The GitLab Duo Agent Platform flow, made of an investigator and a repairer (see `docs/duo-flow.md`) |
| `.gitlab-ci.yml` | The pipeline |
| `docs/contracts/` | JSON schemas for events, evidence bundles and analyses |
| `infra/` | Terraform for the Cloud Run deployment |
| `tests/`, `backend/tests/`, `app/tests/` | The test suites |

## Deploy modes

The deploy mode is set by `SENTINEL_DEPLOY_MODE` in `.gitlab-ci.yml`.

- **`local`** is the default, and the run above used it.
  - The job `canary:local-evaluate` starts two copies of the app: the current commit as the candidate (port 8081), and the previous commit as the stable release (port 8080).
  - The real evaluator sends real requests to both and compares their error rate and p95 latency.
  - This mode needs no cloud account.
- **`cloudrun`** deploys to Google Cloud Run.
  - It builds the image and deploys it as a tagged Cloud Run revision.
  - It shifts traffic 10 → 50 → 100 % and runs an evaluation at each step.
  - If a step breaches its thresholds, it rolls back to the stable revision.
  - It needs a GCP project and a service account (see `docs/deployment_guide.md` and `infra/`).

## Run it locally

You need Python 3.11+ and Node 18+. On Windows, use `127.0.0.1` rather than `localhost`.

```bash
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\Activate.ps1
pip install -r backend/requirements.txt -r app/requirements.txt

# tests
python -m pytest -q backend/tests
python -m pytest -q app/tests
python -m pytest -q tests --ignore=tests/regressions
python scripts/verify.py            # lint + tests + security; ends with [PUSH GATE PASSED]

# backend (http://127.0.0.1:8000)
cd backend && python -m uvicorn app.main:app --port 8000
# dashboard (http://127.0.0.1:5173), in a second terminal
cd frontend && npm install && npm run dev
```

The dev HMAC and webhook secrets in `backend/app/config.py` are public placeholders, for local use only. On GitLab, set `SENTINEL_BACKEND_URL`, `SENTINEL_HMAC_SECRET` and `GITLAB_PROJECT_TOKEN` as masked CI/CD variables.

## Set up the Duo flow on your own GitLab project

1. Go to **Automate → Flows → New flow**. Paste `.gitlab/duo/flows/sentinel-investigate-repair.yml`, then create the flow and enable it.
2. Go to **Automate → Triggers**. Add a trigger for the flow on **Pipeline: Failed**, and choose a service account with the Developer role.
3. Apply `docs/demo/fault-discount-lookup.patch`, push it as a normal commit, and watch the loop.

## Honest limits

- The recorded run used the `local` deploy mode. The Cloud Run path is implemented and unit-tested with a fake `gcloud`, but it was not part of the recorded run.
- Some panels on the System page (saboteur, candidates, shadow replay, carbon) show simulated, reference or derived values. Each panel is labelled with its data source, and no unmeasured value is presented as measured.
- The Duo analysis is advisory. Only the gates count as proof.

## Team

- Krishna Singh: backend, integration, GitLab CI and the Duo flow
- Samvit Srivastava: frontend and AI
- Kailash Ambadipudi: CI/CD and cloud

## License

MIT. See [LICENSE](LICENSE).
