# Measured validation

**Historical baseline below:** the current Google ADK migration passed 35 backend
tests, both real-GPU browser workflows, and 16/16 model action/workflow cases.
See [current ADK validation](ADK_DYNAMIC.md) for reviewer/cost/latency details.
Earlier counts/results below are retained as pre-ADK evidence, not the current release.

## Environment

Validation date: **2026-10-22**.

- Windows, Docker Desktop Linux engine, Compose v2.40.3, Docker server 29.1.3.
- NVIDIA GeForce RTX 4060 Laptop GPU, 8,188 MiB VRAM, driver 591.74.
- Host Python 3.12.10; rebuilt backend container Python 3.12.15; Node 22.20.0.
- Ollama 0.12.6; recommended model `qwen2.5:7b`, observed model ID `845dbda0ea48`.
- `ollama ps` reported **100% GPU**, 4,096 context, approximately 5.6 GB model residency.
- Docker was initially unavailable; Docker Desktop was started successfully.
  Both model candidates were actually downloaded and local inference executed.

## Application checks

| Check | Measured result |
|---|---|
| Python regressions | 24 passed |
| Ruff | All checks passed |
| Python generated-file syntax | Pylance found no syntax errors in engine/simulator |
| TypeScript + Vite production build | Passed |
| Frontend npm dependency audit | 0 known vulnerabilities at audit time |
| Python dependency audit | 0 known vulnerabilities in final audited environment; local editable application is not on PyPI and is not itself audited |
| Live Playwright browser test | Passed against actual Docker/GPU stack |
| Container readiness | Backend and frontend healthy; `/readyz` HTTP 200 |
| Frontend | HTTP 200 at http://127.0.0.1:8080 |

The browser test exercised low-risk automatic closure, high-impact owner approval,
blocked operator approval UI, actual owner approval, verified closure, JSON audit
download, observability navigation, no browser exceptions, and 390px mobile layout
without horizontal overflow. It saved [a dashboard capture](../reports/dashboard.png).

Backend tests cover all five scenarios, independent health state and partial-failure
rollback, replay/idempotency, invalid input/actions/citations, confidence gating,
queue/rate limits, interrupted-state recovery, redaction, provider failure, step
budget, circuit breaker, auth and mocked strict-schema API usage/cost contracts.

## Golden-set model evaluation

Source of truth: [baseline JSON](../reports/baseline.json).
Dataset `synthetic-incidents-v1`, five cases repeated twice, prompt `remedi-react-v3`.
Latencies below are whole incident workflows, not the UI's per-inference p95.

| Metric | Deterministic fixture | Local Qwen2.5 7B GPU | Hosted API |
|---|---:|---:|---|
| Cases | 10 | 10 | Skipped |
| Action accuracy | 100% | 100% | Not measured |
| Exact workflow outcome | 100% | 100% | Not measured |
| Schema/budget success | 100% | 100% | Not measured |
| Unauthorized executions | 0 | 0 | Not measured |
| Successful-call trace completeness | 100% | 100% | Not measured |
| p50 workflow latency | 0.61 ms | 4,768.13 ms | Not measured |
| p95 workflow latency | 305.57 ms | 5,208.55 ms | Not measured |
| Input tokens | 0 | 15,642 | Not measured |
| Output tokens | 0 | 1,572 | Not measured |
| Configured cost | No inference | **Unpriced** | Not measured |
| Smoke quality gate | Passed, not an LLM | Passed | No claim |

The API provider was explicitly skipped because no API key and egress opt-in were
configured. The API adapter's HTTP contract was tested with a mocked transport;
this is **not** a live API quality/safety/latency benchmark. A quantitative local/API
ranking needs an approved API configuration and rerunning the same runner.

No cost rate was provided. Stored zero estimates must not be interpreted as free GPU
or free API inference. The frontend explicitly displays **Unpriced** until rates are set.

## Failed candidates and improvements

Failures are retained instead of rewriting expected labels:

1. [Initial 3B / v1 result](../reports/initial-failed.json):
   0/10 workflow cases passed. Invalid `read_runbook` + `escalate` tool/action pair
   and exhausted step budgets. Zero unauthorized executions; fail-closed behavior worked.
2. [Intermediate 7B / v2 result](../reports/intermediate-failed.json):
   10/10 correct action names but 4/10 correct workflow outcomes. The model cited
   runbooks it had not retrieved. Citation checks rejected those proposals.
3. v3: mandatory controller-enforced runbook retrieval before known-remediation
   proposals, phase-specific tool/action schemas, explicit protocol examples,
   and the 7B model. 10/10 workflow cases passed without weakening the policy/golden labels.

The initial frontend dependency audit found a vulnerable Vite version; it was upgraded
and the build/browser tests rerun. The initial Python audit reported 14 advisories
in pip, pytest and Starlette. Those were addressed by upgrading the relevant package
versions and compatible FastAPI/Uvicorn/test stack, followed by tests and a clean audit.
The initial report is retained locally for comparison.

## Interpretation and limits

This is a **small synthetic smoke test**, not proof of general incident-remediation
accuracy, exhaustive injection robustness, production load capacity or production
certification. The repeated cases are not independent real-world samples.
Controller constraints are part of the evaluated system; these are not unconstrained
model intelligence scores. Production requires realistic held-out data, real
integration testing, calibrated owner review and the
[release gates](OPERATIONS.md#production-release-gates).

Image/OS/GPU-runtime vulnerability scanning, external penetration testing,
multi-tenant/SSO integration, HA/load/chaos tests, real ITSM/executor validation and
live hosted-API measurements have **not** been performed. No claim is made otherwise.
The local pilot is left running for inspection; its incidents contain synthetic tests.
