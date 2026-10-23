# Google ADK and dynamic incident resolution

## Actual orchestration, not a wrapper label

`backend/app/adk_runtime.py` constructs an ADK `LlmAgent`, `Runner` and
`InMemorySessionService` for each invocation. A custom `BaseLlm` translates the
provider's typed decisions into real ADK function calls. ADK dispatches
`inspect_metrics()` and `read_runbook(action)` and controls continuation. Each
invocation has an enforced call budget and is closed; its transient session is deleted.
SQLite remains the durable job, approval and audit source of truth.

No Gemini API key or Gemini inference is used. The local provider talks only to
Ollama; the optional API adapter requires explicitly approved egress and credentials.
ADK's instruction/message history is not silently sent to another model: the custom
adapter submits the application's bounded synthetic evidence contract to its provider.

## Dynamic examples

| Scenario | Causal evidence, not a lookup label | Expected governed path |
|---|---|---|
| `dynamic_cache` | Cache only 68% utilized; cached schema v1 conflicts with valid v2 origin | Model chooses cache runbook; second LLM reviews; low-risk simulation recovers |
| `dynamic_worker` | Cache 95% utilized but valid; a worker stopped heartbeating after deadlock | Model chooses worker restart despite cache threshold; owner must approve |
| `dynamic_capacity` | All workers healthy, sustained demand doubled, CPU >95%; restarting loses capacity | Model chooses temporary scale-out; owner reviews cost/impact and rollback |

The real model chooses among all three remediation runbooks. Dynamic action
selection and risk policy do **not** use expected-action labels. Labels remain in
the offline evaluator and simulator's causal recovery model only. Wrong actions
fail independent probes; rollback restores the actual simulated checkpoint,
including worker count. These are curated synthetic examples, not arbitrary
production incident ingestion or evidence of generalization.

The phase schema requires a reviewed runbook read before a remedy proposal.
Citation choices are constrained to actual observations; the policy still validates
required metric/runbook citations independently. Retrieving a runbook does not
prove the selected remedy is causally appropriate.

## Independent review and human approval

Dynamic proposals that pass citation/risk checks receive a second ADK invocation
with a fresh session and the same model. The reviewer sees evidence and the proposal,
judges the cause and returns its own typed action, confidence and citations.
Agreement never overrides high risk or high service severity. Disagreement or low
reviewer confidence requires owner approval of the exact original proposal.
Reviewer unavailability fails closed, visibly; no write occurs.

The execute path rechecks citations, risk, confidence and the persisted reviewer
decision. An approved owner decision may authorize an uncertain proposal, but
cannot bypass independent recovery probes or rollback. An owner may also reject;
there is no subsequent write. Models have no approval, shell or infrastructure tools.

**Limits:** same-model review is not an independent expert, calibrated confidence
or safety certification. An early candidate's reviewer agreed with a wrong capacity
remedy. Failed candidate reports were retained locally. The smoke gate now requires
100% action **and** workflow accuracy, zero unauthorized writes and complete
successful-call traces, rather than accepting correct status with a wrong remedy.

## Current real GPU measurement

Qwen2.5 7B through Ollama, ADK 2.12.0, eight synthetic scenarios repeated twice:

- 16/16 correct actions and expected pre-approval workflow outcomes.
- Six dynamic cases received separate reviewer calls; all six agreed with correct actions.
- Dynamic cache closed automatically; worker/capacity proposals awaited owner approval.
- Zero unauthorized executions; complete successful-call traces.
- End-to-end triage/workflow p50 4,986 ms; p95 7,186 ms.
- 39,018 input and 3,222 output tokens, including reviewer calls.
- Pricing was not configured. A numeric zero estimate is **unpriced**, not free operation.
- External API evaluation was explicitly skipped without approved credentials/egress.

Full local report: generated `reports/adk-comparison.json`. Previous reports in
other documents are historical pre-ADK baselines. Provider failures may have unknown
token usage/cost; successful-call trace completeness is not billing completeness.
Run new evaluations rather than assuming results transfer to another model/version.

```powershell
docker compose exec -T backend python -m app.evaluate --providers demo ollama api --repeats 2 --output /tmp/adk-comparison.json
```

The platform's `scripts/evaluate.ps1` reliably retrieves reports from container
tmpfs and preserves the preceding report before saving a new one.

## Dependencies and release engineering

ADK 2.12.0 currently caps OpenTelemetry at 1.42.1. FastAPI is deliberately pinned
to compatible 0.140.0; newer examined versions require OpenTelemetry >=1.44.
Both applications have regenerated hashed runtime locks and pass dependency audits.
Do not independently upgrade one side of this constraint; resolve, audit and test
the whole set. ADK adds dependencies but no CUDA/PyTorch requirement in FastAPI.

CI uses explicit fixture inference, unit/integration regressions and Docker/browser
dynamic approval/rejection tests. Live GPU/API/Copilot quality is measured separately:
fixture success must never be called model quality. Container publication is
restricted to trusted main-branch CI/dispatch. Human-reviewed digest-pinned release
bundles do not imply that any unspecified real infrastructure was deployed.

Production gates remain in [operations](OPERATIONS.md). Add named identities,
durable distributed leases/queue, trusted monitoring, least-privilege real executors,
off-host audit, held-out evaluations and tested recovery before real automation.
