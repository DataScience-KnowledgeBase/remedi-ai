# Dependencies, integrations and operations

## Dependencies and reproducibility

| Layer | Runtime |
|---|---|
| Backend | Python 3.12+, FastAPI, Pydantic settings, httpx, uvicorn, prometheus-client |
| Validation | pytest/pytest-asyncio, Ruff; isolated `.venv` |
| UI | Node 22.20+, React 19, TypeScript 5.8, Vite 7; npm lockfile and `npm ci` |
| Serving | Non-root Python image and unprivileged nginx |
| Inference | Ollama 0.12.6, `qwen2.5:7b`, 4K context, one loaded model/parallel inference |
| Storage | SQLite WAL and Docker named volume; one backend process/worker only |

Direct and transitive Python runtime versions are pinned in `backend/requirements.lock`,
and npm has its package lock. The backend image installs the runtime lock before the
application with `--no-deps`; tests should use the same constraints. Python 3.12.15
was observed in the rebuilt image. Base-image tags are not immutable locks.
Before release add verified hashes to the Python lock,
pin images and Ollama model artifacts by reviewed digest, generate SBOMs, verify model
license/model-card provenance, scan images, and sign artifacts. Qwen2.5 7B is an
Apache-2.0 model; independently review downloaded artifact and redistribution terms.
The initial 3B candidate has different licensing considerations and was rejected by
the workflow evaluation; it is not the recommended deployment.

No host PyTorch/CUDA dependency is required. Docker Desktop must expose the GPU via WSL2.
Do not install Linux NVIDIA drivers inside the Windows WSL environment indiscriminately;
follow Docker/NVIDIA's current GPU guidance for the installed platform.

```powershell
docker info
docker compose ps
docker compose exec -T ollama nvidia-smi
docker compose exec -T ollama ollama ps
docker compose logs --tail 80 backend
```

The model is downloaded into the project-owned `ollama-models` volume, not bundled
in the backend image. Do not mount the Docker socket or host filesystem into the agent.

## Operations and service limits

- UI: `127.0.0.1:8080`; backend/Ollama have no published host ports.
- `/healthz`: nginx/app liveness; `/readyz`: worker alive + local model installed.
  API-provider readiness verifies configuration, not remote availability; inference
  and error metrics reveal actual provider health.
- One worker bounds GPU pressure. Queue admission defaults to 25 active incidents,
  including approval waits; the operator receives `409` when full.
- Authenticated traffic is limited to 120 requests/minute **per shared role** in-process.
  Invalid tokens are not covered by that limiter: production ingress must enforce
  IP/client and unauthenticated brute-force limits. No distributed limiter is claimed.
- SSE is capped at 20 minutes per connection; reselect the incident to reconnect.
- No prompt response caching: stale incident evidence must not be reused.
- Read-only backend root filesystem; SQLite writes only to `/data`; `/tmp` is tmpfs.
- Use `docker compose stop` to stop this project. Do not use global container cleanup.
  Avoid `down -v`: it destroys incident/model volumes.
- `.env` tokens have role identity, not individual human identity. Rotate by editing
  secrets and recreating backend; connected streams end on recreation.

## Backup and disaster recovery

Use SQLite's online backup API for a consistent live snapshot. Never copy only the
main DB file while WAL writes are active:

```powershell
docker compose exec -T backend python -c "import sqlite3; s=sqlite3.connect('/data/remedi.db'); d=sqlite3.connect('/data/remedi-backup.db'); s.backup(d); d.close(); s.close()"
docker compose cp backend:/data/remedi-backup.db .\remedi-backup.db
```

Protect backups as incident/audit data. Encrypt storage, restrict ACLs, offload to
approved storage with retention/WORM policies, and verify restore in an isolated project.
To restore: stop backend, restore the reviewed database into its volume, verify
`PRAGMA integrity_check`, then start backend and review interrupted workflows.
Never delete or overwrite another project's volume. Define/test RPO and RTO with
the service owner; neither HA nor multi-region failover is implemented here.

## Integration contracts

| Integration | Current implementation | Required production adapter |
|---|---|---|
| Monitoring/incident ingestion | Typed synthetic scenarios only | Signed authenticated webhook; dedup source event ID; canonical severity/asset ownership; timestamp/freshness checks; schema/PII validation |
| Evidence | Synthetic metrics with source ID | Read-only least-privilege metrics/log connector; immutable provenance and freshness window; no untrusted severity overrides |
| Runbooks | Versioned exact in-process catalog | Reviewed signed registry, ownership, blast radius, prerequisites, verifiable success/rollback tests |
| Executor / Presto from diagram | In-memory checkpoint/effects plus persisted audit | Privilege-separated service, scoped credentials, target allowlist, dry-run, change/maintenance window, durable checkpoint, idempotent job ID and rollback |
| Verification | Three synthetic threshold probes | Independent monitoring account and delayed repeated probes; all critical dependencies, SLOs, customer outcomes and stale-data failure |
| ITSM | Persisted local incident + exported audit | ServiceNow/Jira API, approved mappings, change ticket linkage and closure only after observed recovery |
| Identity | Two distinct role tokens | OIDC/SSO, MFA, named identities, tenant isolation, approver ownership, separation of duties and short-lived claims |
| Learning | Outcome review candidates | Human review, held-out replay, canary, signed version promotion; never self-modifying production runbooks |
| Observability | Audit JSON/SSE, Prometheus endpoint | Authenticated scraper, encrypted remote log/trace storage, paging and retention controls |

## Observability and cost

Every successful model call records provider/model, prompt version, token counts,
latency, attempts and configured estimated USD. Audit spans are correlated by incident ID
and global sequence. Prometheus exposes inference histograms, input/output tokens,
configured cost totals, workflow outcomes and failures at authenticated `/api/metrics`.
Use an operator credential via a scraper secret file, never a URL query token.
The pilot does not implement OpenTelemetry exporters or a preinstalled Grafana stack.

Recommended alerts: worker/provider readiness failure, inference failure/circuit open,
elevated p95, stuck queue/approval age, rollback, unexpected escalation, token/cost
budget, drift in outcome distribution. Counters reset on process restart; retain
scrapes remotely. UI telemetry covers the most recent 100 incidents, not all-time billing.

All price defaults are zero **unconfigured**, not free. API estimates use provider
reported tokens and configured per-million rates. Local estimates allocate inference
wall time at the configured GPU/hour rate; include idle time, model download/load,
power, hardware amortization and operational labor in TCO separately. Failed requests
may incur unknown token charges that this app cannot infer. See [evaluation](EVALUATION.md).

## Production release gates

1. Integrate SSO/named accountable approvers, MFA and immutable external audit.
2. Enable TLS at approved ingress, private networking, egress allowlisting, request/body/
   unauthenticated limits, security monitoring and secret-manager rotation.
3. Build trusted evidence freshness/ownership validation and privilege-separated real
   executors. Test rollback against a staging service with independent verification.
4. Review all runbooks with service owners: impact, maintenance window, rate of changes,
   blast radius, execution deadlines and an external emergency kill switch.
5. Expand privacy-reviewed held-out incidents, adversarial payloads and dependency/
   service failure cases. Set actual quality/safety/SLO/cost gates and human review.
6. Lock/hash dependencies, pin/scan/sign images/models, track licenses and SBOMs, scan
   secrets in CI. Run unit, browser, contract, load and recovery tests.
7. Persist backups and logs off-host, test restores, define retention/RPO/RTO and incident ownership.
8. Shadow recommendations first, then owner-approved staging, then a narrowly scoped
   canary. Auto-execution remains disabled for unknown, high-risk or unverifiable actions.
9. For horizontal scale: migrate SQLite to PostgreSQL, durable broker/leased jobs,
   per-tenant identities/quotas, transactional outbox and distributed idempotency.
   Only then introduce Kubernetes replicas, queue/latency autoscaling and GPU scheduling.
10. Independently audit the completed real adapters and threat model before any unattended
    production changes. No synthetic benchmark can replace that review.
