# Local versus API security and observability

## Trust boundaries

The browser has only an operator/owner application token. Provider secrets stay on the
server. A model may recommend, never execute or approve. Trusted scenario signatures
and deterministic policy determine action eligibility. Incident log instructions are
untrusted data; no arbitrary commands, URLs or filesystem tools are available.

| Aspect | Local Ollama GPU | HTTPS model API |
|---|---|---|
| Evidence residency | Inference stays in project network/machine | Synthetic evidence leaves machine only with opt-in; review provider retention/region |
| Model access | Unpublished Docker port; private bridge | Server-only API key, HTTPS, no redirects/proxy inheritance |
| Attack surfaces | Host/GPU/model supply chain, local network, artifacts | API credentials, internet dependency, vendor/region, retention and rate limits |
| Injection risk | Present; local hosting does not make a model safe | Present; hosted models do not replace action policy |
| Deterministic enforcement | Same schema, phase gates, signature/evidence checks, HITL | Same controller and policy, independently from provider |
| Auth and approval | Distinct operator/owner pilot tokens | Same; API model never receives those credentials |
| Availability | GPU memory, driver, thermal/load and model lifecycle | Connectivity, quotas, rate limits, provider outages |
| Observable usage | Ollama prompt/eval counts, local elapsed time, configured GPU rate | Provider prompt/completion usage, elapsed time, configured token rates |
| Traceability | Incident/model/prompt/runbook IDs, bounded tools, audit | Same, plus outbound provider identity; no raw API keys logged |
| Cost interpretation | Zero token invoice does not mean zero TCO | Estimate is not a bill; failed/retried requests may be charged |

## Implemented controls and tests

- Strict typed API and model schemas reject extra input and arbitrary tools.
- Phase-specific schemas prevent skipping required evidence/runbook retrieval.
- Trusted signature mismatch or fabricated citation prevents execution.
- Low confidence and high impact require owner approval; approvals cannot be replayed.
- Both providers operate behind the same deterministic write boundary.
- Queue capacity, authenticated role rate limits, bounded steps, output tokens,
  context, inference timeout/retries and circuit breaker constrain resource use.
- No local-to-remote fallback; API mode requires `ALLOW_API_EGRESS=true` and a key.
- Input is scenario-only, not arbitrary external incidents/documents. Redaction of
  email/common token patterns is defense-in-depth, **not a complete DLP system**.
- Audit payloads are recursively redacted before JSON serialization. Secrets are not
  sent to React, browser storage or SSE URLs.
- Non-root app containers, read-only app rootfs, dropped Linux capabilities,
  `no-new-privileges`, localhost ingress, CSP and security headers.
- Append-only SQLite event triggers prevent application updates/deletions, not
  privileged file tampering. Off-host WORM audit is a production requirement.
- Regression tests exercise auth, permission separation, replay, injection scenario,
  untrusted citations, invalid schemas, rate limits, fail-closed provider error and circuit.
- API adapter tests use an in-process mocked HTTP transport: they validate the client
  contract **without** a network call or claiming a hosted model's safety score.

## Limits and deployment obligations

This is not a penetration test or security certification. It does not implement
OIDC/MFA, tenant isolation, per-user accountability, encrypted database fields,
WORM audit, comprehensive PII detection, gateway brute-force protection, DDoS control,
signed webhooks, image signing or a real privileged executor. Those are explicit
[release gates](OPERATIONS.md#production-release-gates), not implied features.

Prompts alone are not an injection defense. A compromised model can degrade availability
or propose bad actions; the controller's signature/risk/evidence checks must remain
authoritative. When adding real incident ingestion, build canonical trusted metadata
and provenance separately from untrusted logs. Never permit the LLM to supply its
own severity, ownership, approval identity or allowed target.

Initial frontend dependency audit flagged the original Vite version. It was upgraded
within a supported Node runtime and the production build was rerun; see the measured
validation report for the final audit. Re-run npm/Python/image audits at each release;
a clean point-in-time npm result says nothing about unscanned base images or future CVEs.
