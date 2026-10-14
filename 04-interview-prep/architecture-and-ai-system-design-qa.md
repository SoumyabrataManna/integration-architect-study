# Architecture and AI System Design Q&A

Company-neutral system-design questions for senior integration and solution architect interviews.

## Section 1: API and platform design

### Q1. Design a multi-tenant API gateway
- **AuthN/Z:** OAuth2 with JWT carrying tenant claims; mTLS for service-to-service; validate signature, `exp`, `aud`, `iss` at the gateway.
- **Rate limiting:** per-tenant sliding window in Redis, key `rate:{tenant_id}:{endpoint}:{window}`; return 429 with `Retry-After`.
- **Versioning:** URL versioning (`/v1`), support N-1, publish `Deprecation` and `Sunset` headers.
- **Resilience:** circuit breaker per upstream (for example open at >50% errors over 10s with a minimum of 20 requests), bulkheads, timeouts.
- **Isolation:** tenant ID mandatory on every downstream call and query; per-tenant quotas and audit logs.

### Q2. Make an API idempotent
- Client sends an `Idempotency-Key` (UUID) header on POST.
- Server stores key and state in Redis or a table: `PROCESSING` then `COMPLETED` with the stored response (two-phase, so concurrent retries wait or get 409).
- Retain keys about 24 hours.
- PUT and DELETE are idempotent by definition; make PATCH idempotent by sending absolute values, not deltas.

### Q3. Design a real-time analytics API
- Batch ingest endpoint returning 202 Accepted.
- Kafka topic partitioned by `user_id` to preserve per-user ordering.
- Server-Sent Events (or WebSocket) for live dashboards.
- Avro or Protobuf with a schema registry and BACKWARD compatibility.

## Section 2: AI platform and data design

### Q6. Design a multi-tenant RAG platform
1. **Ingestion:** parse, chunk (about 512 tokens, 25% overlap), embed, store with metadata (tenant, source, ACL).
2. **Storage:** per-tenant vector namespaces plus a keyword index.
3. **Retrieval:** hybrid search (BM25 plus dense vectors) merged by Reciprocal Rank Fusion, then cross-encoder re-ranking; enforce document-level permissions at query time.
4. **Generation:** temperature 0 for factual tasks, answers must cite sources, refuse when evidence is missing.
5. **Governance:** audit log of prompts, retrieved chunks and answers; right-to-deletion that removes vectors, chunks and caches.

### Q7. Design a Customer 360
- Capture changes with CDC (for example Debezium) into an event stream.
- Identity resolution: deterministic rules first (email, tax ID), probabilistic matching second, with a steward review queue for low-confidence matches.
- Build a golden record with survivorship rules.
- Medallion layers: bronze (raw), silver (cleaned, conformed), gold (business views).
- Define freshness SLAs per consumer.

## Section 3: AI strategy and evaluation

### Q11. RAG vs fine-tuning vs prompt engineering

| Approach | Use when | Strengths | Limits |
|---|---|---|---|
| Prompt engineering | Behaviour or format needs steering | Fast, cheap, reversible | Limited by context and model knowledge |
| RAG | Answers need fresh, private or citable knowledge | Up-to-date, auditable, no retraining | Retrieval quality is the ceiling |
| Fine-tuning | Consistent style, domain behaviour or latency/cost reduction | Bakes in behaviour | Stale knowledge, retraining cost, harder governance |

Typical answer: start with prompting, add RAG for knowledge, fine-tune only when evaluation shows a remaining behavioural gap.

### Q12. Evaluate an AI feature in four levels
1. **Offline:** golden dataset with metrics such as RAGAS (faithfulness, answer relevance, context precision/recall).
2. **Shadow mode:** run on live traffic without showing users; compare to current process.
3. **A/B test:** measure task success, acceptance rate, time saved.
4. **Continuous monitoring:** drift detection (for example PSI > 0.25 triggers review), LLM-as-judge sampling, user feedback loop.

## Section 4: Integration architecture

### Q15. Choose an integration platform (iPaaS vs custom event streaming)
Compare on: time to deliver, connector coverage, skills, governance and audit, run cost, vendor lock-in, scale ceiling. Principles: decouple with events where latency allows, keep contracts explicit and versioned, centralise cross-cutting concerns (security, logging), own your canonical model.

### Q16. Use AI inside integration workflows
- **Mapping suggestion engine:** the model proposes source-to-target field mappings with a confidence score. Below 70% routes to human review; at or above 70% auto-accepts into a draft, still subject to validation tests.
- **Architecture validation:** AI reviews designs against standards (error handling, idempotency, security) and flags gaps.
- Keep a human approval gate and log every AI decision.

### Q17. Build resilience into integrations
Exponential backoff with jitter; dead-letter queues; circuit breakers; idempotent consumers; timeouts and bulkheads; observability (correlation IDs, metrics, alerts, runbooks).

### Q18. Integrate systems after a merger or acquisition
- Avoid point-to-point (N² connections); use an event hub or API layer.
- Define a canonical data model and map each system to it once.
- Use the **strangler fig** pattern to migrate incrementally.
- Use dual-write or change-capture during transition, controlled by feature flags, with reconciliation reports before cutover.

### Q19. MCP servers and AI-assisted development
- MCP exposes tools, resources and prompts to AI clients through a standard protocol.
- **Context engineering:** project context files (for example a `claude.md`) capture standards, data models and conventions.
- **Confidence scoring** on AI outputs, with human review below a threshold.
- **Production safety for AI-generated code:** code review, tests, static analysis, least-privilege credentials, staged rollout.

## Section 5: Behavioural and prioritisation

- **STAR:** Situation, Task, Action, Result. Keep the Result quantified.
- **Prioritising AI investments:** score each idea on ROI (value, cost, effort, risk), plot on a 2x2 (impact vs effort), and balance the portfolio roughly 70% core improvements, 20% adjacent bets, 10% experiments.

## Section 6: Questions to ask the interviewer
- How are integration standards defined and enforced across teams?
- What does the current integration landscape look like, and what is the biggest pain point?
- How is AI currently used in delivery, and what guardrails exist?
- How do you measure integration success (SLAs, MTTR, reuse)?
- What does a successful first 90 days look like?

### Pre-interview checklist
- [ ] Four stories prepared with metrics
- [ ] One system design rehearsed end to end (API, events, data, security)
- [ ] Idempotency, ordering and failure-recovery answers ready
- [ ] Platform-specific details reviewed (runtime, error handling, governance)
- [ ] Questions for the interviewer written down
