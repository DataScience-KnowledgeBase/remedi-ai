# RemediAI

Governed incident remediation based on the supplied **AI_Remedy_1** and
**AI_Remedy_2** diagrams and the contents of the **Section1–Section19** notebooks.

**React** is the frontend framework. **ReAct** is the bounded
reason/action/observation control loop. **Google ADK 2.12.0** now owns that loop
through a real `LlmAgent`, `Runner` and native read-only function tools.
See [ADK and dynamic remediation](docs/ADK_DYNAMIC.md) for the current architecture,
examples, measured results and limitations.

## What is implemented

- React/TypeScript operations console: incident injection/filtering, live authenticated
  SSE trace, evidence inspection, accountable owner approval/rejection, audit export,
  runbook catalog, feedback candidates, latency/tokens/cost/outcome telemetry.
- FastAPI backend with validated requests/model responses, durable SQLite queue,
  idempotency, distinct operator/owner roles, bounded inference retries, timeout,
  circuit breaker, rate limits, queue limits and crash recovery.
- Allowlisted ADK read-only tools; deterministic legacy-signature/risk/evidence policy
  controls every simulated write. The LLM never receives shell access or approval authority.
- Automatic low-risk recovery; higher impact/low confidence approval gate; three
  independent health samples before closure; checkpoint restoration on failed verification.
- Local Ollama GPU inference with Qwen2.5 7B and an opt-in OpenAI-compatible API adapter.
  Deterministic `demo` mode is explicitly **not** an LLM benchmark.
- Non-root Docker application containers, persistent volumes, health probes,
  localhost-only ingress, and offline golden-set evaluations and security regressions.

> **Deployment scope:** this is a working, production-oriented **single-instance
> simulation/pilot**, not certification for unattended real infrastructure changes.
> All incidents and remediation effects are synthetic. Real monitoring, ITSM,
> identity, privileged execution and recovery integrations require the gates in
> [the production checklist](docs/OPERATIONS.md#production-release-gates).
> Do not expose the current shared-token pilot directly to the public internet.

## Quick start — Windows + Docker Desktop + NVIDIA GPU

Requirements: Docker Desktop Linux engine with WSL2 GPU passthrough, NVIDIA driver,
Docker Compose v2. An RTX 4060 Laptop GPU with 8 GB VRAM was detected on this machine.
The validated 7B model occupies approximately 5.6 GB at 4,096 context tokens.
Allow at least 15 GB additional disk space for images/model caches and ~12 GB host RAM.
No CUDA/PyTorch install is needed in the FastAPI environment: Ollama owns GPU inference.

From this folder in PowerShell:

```powershell
.\scripts\bootstrap.ps1
docker compose --profile gpu up -d --build
docker compose exec -T ollama ollama pull qwen2.5:7b
docker compose exec -T ollama ollama ps
Invoke-RestMethod http://127.0.0.1:8080/readyz
```

Open **http://127.0.0.1:8080**. Open the generated private `.env` locally in your editor:
use `OPERATOR_TOKEN` to connect and create incidents; switch to `OWNER_TOKEN`
to approve/reject high-impact incidents. Tokens remain in browser memory only.
Do not share or commit the private file. Bootstrap does not overwrite an existing one.

Try these scenarios:

| Scenario | Expected path |
|---|---|
| Dynamic schema mismatch | LLM causal diagnosis below cache threshold -> independent LLM review -> low-risk recovery |
| Dynamic worker deadlock | LLM distinguishes healthy cache from worker fault -> review -> owner approval |
| Dynamic demand saturation | LLM recommends capacity, not restart -> review -> owner approves cost/impact |
| Cache pressure | Triage → mandatory runbook read → policy → execute → verify → close |
| Worker stalled | Triage → recommend → owner approval → execute → verify → close |
| Unknown signal | Escalate; no write |
| Recovery failure | Execute → failed health probes → restore checkpoint → escalate |
| Untrusted signal | Treat injected log as data → escalate; no write |

The frontend uses a streamed trace, not synthetic progress timers. Expanding an event
shows evidence, model/prompt identity, usage and timing. Export audit JSON for review.
The trace contains concise operational explanations, **not private chain-of-thought**.

### No GPU / deterministic demonstration

On a fresh setup: `.\scripts\bootstrap.ps1 -Provider demo`, then
`docker compose up -d --build`. For an existing setup deliberately edit `PROVIDER=demo`
and recreate the backend. Do not label fixture output as LLM quality or latency.
CPU Ollama is possible with a separate CPU-only deployment, but is not silently substituted.

### API option

The adapter is implemented, not placeholder pseudocode. Commented settings in
`.env.example` show the alternative:

```dotenv
# PROVIDER=api
# ALLOW_API_EGRESS=true
# API_BASE_URL=https://api.openai.com/v1
# API_MODEL=gpt-4.1-mini
# API_KEY=<secret-manager-injected-key>
# INPUT_USD_PER_MILLION=<verified-current-price>
# OUTPUT_USD_PER_MILLION=<verified-current-price>
```

Enable these only after reviewing provider retention, region, contractual data handling
and egress policy. Strict JSON-schema chat-completions support is required; not every
"OpenAI-compatible" endpoint supports this contract. Recreate backend with
`docker compose up -d --build --no-deps backend`. Never place API keys in React code.
There is **no automatic local-to-API fallback** or unapproved external egress.
Only synthetic scenario data is accepted by this pilot.

## Evaluate and validate

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install --upgrade pip==26.2
.\.venv\Scripts\python.exe -m pip install -c .\backend\requirements.lock -e ".\backend[dev]"
.\.venv\Scripts\python.exe -m pytest .\backend\tests -q
.\.venv\Scripts\ruff.exe check .\backend
.\.venv\Scripts\python.exe -m pip_audit --local

Set-Location .\frontend
npm ci
npm run build
npx playwright install chromium
npm test
```

The browser test uses this project's private `.env` without printing tokens.
It tests actual deployed-provider recovery, the owner gate, audit download,
observability, browser errors and mobile overflow. Run against a disposable pilot
database if you do not want synthetic test incidents retained.

Run an isolated dataset comparison inside Docker:

```powershell
docker compose exec -T backend python -m app.evaluate --providers demo ollama api --repeats 3 --output /data/comparison.json
docker compose cp backend:/data/comparison.json .\reports\comparison.json
```

Evaluation uses a temporary database and never changes operational incidents.
Exit `1` means a completed provider failed the quality gate; `2` means no provider
could run. Skipped providers are explicit. The API is not benchmarked without a key
and egress opt-in. See [evaluation methodology](docs/EVALUATION.md) and
[measured results](docs/VALIDATION.md).

## Project layout

| Path | Purpose |
|---|---|
| `backend/app/domain.py` | Golden synthetic signatures, policy, typed model output, runbooks |
| `backend/app/providers.py` | Local/API adapters, phase-specific tool schemas, usage and circuit breaker |
| `backend/app/engine.py` | Bounded ReAct, policy gate, simulator, verification and learning |
| `backend/app/store.py` | Transactional persistence, append-only audit, durable queue |
| `backend/app/main.py` | Authenticated REST/SSE and Prometheus telemetry |
| `backend/app/evaluate.py` | Reproducible provider comparison and quality gate |
| `frontend/` | React UI, nginx ingress, browser workflow test |
| `compose.yaml` | Isolated project services and GPU profile |
| `scripts/` | Private credential bootstrap and validation commands |
| `reports/` | Generated measured evaluation evidence |
| `docs/` | Architecture, notebook mapping, operations, security and evaluation |

API schema: **http://127.0.0.1:8080/api/docs**. Authenticated endpoints require a
Bearer header; SSE uses authenticated fetch rather than putting a token in a URL.

## Documentation

- [Architecture and notebook traceability](docs/ARCHITECTURE.md)
- [Dependencies, integrations and operations](docs/OPERATIONS.md)
- [Local/API security and observability comparison](docs/SECURITY.md)
- [Evaluation, metrics and cost methodology](docs/EVALUATION.md)
- [Measured validation and known limits](docs/VALIDATION.md)

The diagram's business-value figures (300+ incidents/month, 200 manual hours,
60% reduction, ~120 hours saved) are **targets from the reference**, not measured
outcomes of this synthetic pilot.
