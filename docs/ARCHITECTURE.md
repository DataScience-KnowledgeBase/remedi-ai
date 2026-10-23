# Architecture and reference traceability

**Current orchestration:** Google ADK `Runner`/`LlmAgent` dispatch native metrics
and runbook tools through a custom `BaseLlm` provider adapter. Dynamic cases add a
fresh reviewer invocation before the deterministic decision. See
[ADK migration and dynamic examples](ADK_DYNAMIC.md); earlier measured baselines
in other documents predate this migration.

## Source interpretation

Both supplied diagrams were visually inspected. Markdown content from Section1
and Sections2–19 was examined; this design is not inferred from filenames alone.
The supplied Section18 attachment describes GPU scheduling, containers, queues,
autoscaling, multi-region failover and cost optimization, and explicitly distinguishes
simulated infrastructure from real cloud deployments. This project preserves that distinction.

| Reference content | Applied design and deliberate limits |
|---|---|
| Section1: ML lifecycle, pipelines, reproducibility, evaluation, drift | Versioned synthetic data/signatures, prompt/runbook versions, golden labels, repeatable smoke gates and reviewed feedback |
| Sections2–9: model/token/inference/training trade-offs | Quantized GPU model, 4K context, 512 output tokens, deterministic sampling, measured model selection; no unjustified fine-tuning |
| Sections10–11: RAG provenance, exact IDs, permissions, retrieval evaluation | Exact allowlisted runbook lookup with observed citation IDs; three documents do not warrant an embedding/vector database |
| Section12: ReAct, state, memory, tools, HITL, guardrails, recovery | Bounded read-only tool loop, persisted incidents/events, deterministic action policy, explicit failures |
| Section13: specialized responsibilities, supervisor, handoff | Separate triage/model, policy, execution simulator, verification and feedback responsibilities; one supervised workflow, not unnecessary autonomous agent processes |
| Section14: async FastAPI, schemas, auth, quotas, durable jobs, idempotency | Async inference/SSE, Pydantic, roles, rate/queue limits, transactional SQLite queue, replay-safe approval |
| Section15: golden sets, task/tool quality, citations, regression, release gates | Dataset hash, action/workflow correctness, citation precision, unauthorized actions, trace completeness, p50/p95 and repeated runs |
| Section16: injection, least-privilege tools, PII, supply chain, governance | No model write tools, no shell, strict output schemas, role gates, redacted audit fields, egress opt-in, CSP, dependency audit |
| Section17: prompt/model identity, canaries, tracing, cost, rollback | Persisted step telemetry, Prometheus counters/histograms, retained failed candidate evaluations, documented promotion and rollback |
| Section18: containers, GPU isolation, scaling, capacity/DR/cost | GPU Compose profile, bounded single concurrency, persistence/backup, capacity limits; no fictitious autoscaling or multi-region HA |
| Section19: requirements, architecture, trade-offs, scale, security | Explicit pilot boundary, integration contracts, release checklist and local/API comparison |

## Control plane

```text
React console
  -> nginx (same-origin HTTP/SSE; localhost binding)
    -> FastAPI (role -> rate limit -> validation -> queue/idempotency)
      -> SQLite incidents + append-only audit + feedback
      -> one durable-queue consumer
        -> Google ADK bounded ReAct triage
          -> inspect_metrics [synthetic]
          -> read_runbook [allowlisted exact lookup]
          -> propose [JSON schema]
        -> separate ADK LLM review for dynamic cases
        -> deterministic policy recheck
          -> unsupported/untrusted/no evidence -> escalation
          -> high impact / low confidence -> owner approval
          -> cited + reviewed + low risk + reversible + verifiable -> auto action
        -> simulator checkpoint -> execute -> three independent probes
          -> healthy -> close
          -> unhealthy -> restore checkpoint + owner escalation
        -> outcome -> reviewed runbook candidate (never auto-promoted)

Model boundary:
  Ollama -> quantized Qwen2.5 7B -> NVIDIA GPU
  OR explicit opt-in -> HTTPS strict-schema API
```

The phase schema requires metrics first. A recognized signature requires runbook
retrieval before a proposal; reading the runbook is a **controller invariant**,
not something a probabilistic model may skip. The LLM still selects a runbook/action,
generates a justification and cites observations. Deterministic policy checks the
legacy signature or dynamic risk boundary, and exact citations before execution.
Dynamic policy does not compare the model action to offline expected labels.
Model confidence is uncalibrated
self-report; it can restrict automation but never override risk policy.

This is a constrained workflow agent, not open-ended autonomy. Operational descriptions
are logged; hidden reasoning is neither requested nor exposed. No internet browsing,
shell execution, code generation, arbitrary URLs, or arbitrary incident text is accepted.

## State, persistence and failures

`queued -> triaging -> ready | awaiting_approval | escalated | failed`

`awaiting_approval -> approved | rejected`

`ready | approved -> executing -> verifying -> closed | rolled_back`

- Status changes and their audit events commit atomically.
- The unique idempotency key binds to the scenario; reuse with different input is `409`.
- A status precondition prevents approval replay. Owner approval binds to the exact
  persisted immutable proposal. Operator and owner tokens must differ.
- Queued and approved jobs survive restart. Mid-triage/execution/verification
  interruption becomes an explicit failure requiring investigation; writes are not blindly retried.
- Model retries are read-only, bounded to two transport attempts. Three consecutive
  inference failures open the circuit for 30 seconds. Each agent gets at most four steps.
- Provider errors and exhausted budgets fail closed, visibly. Unexpected worker failure
  makes readiness fail rather than reporting a healthy worker.
- SQLite triggers reject in-application event updates/deletes. This is not tamper-proof
  against a privileged database/file administrator; production needs WORM/remote audit.
- Async SSE reconnect/replay is supported by `after=<last sequence>`; the current UI
  replays the trace when an incident is reselected. It displays disconnections explicitly.

## Why constrained ADK rather than open-ended autonomy or a training stack?

There are only three reviewed runbooks. Exact keyed retrieval is easier to audit,
cheaper and more reliable than speculative semantic retrieval. The simulator has
known typed inputs, so an ML classifier trained on eight fixtures would be leakage,
not evidence of generalization. Fine-tuning requires a demonstrated gap and a privacy-
reviewed held-out dataset; prompting/control changes are evaluated first.

One GPU and SQLite intentionally imply one worker/replica. Horizontal scaling requires
PostgreSQL, queue leases, transactional outbox, idempotent real executors and distributed
quotas. Do not add replicas to this implementation and assume distributed safety.
See [operations](OPERATIONS.md) for the migration path and release gates.
