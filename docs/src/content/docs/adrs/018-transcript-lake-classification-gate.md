---
title: "ADR-018: Governed transcript lake with a classification-first privacy gate"
description: A proposed direction for a transcript lake behind a classification gate. Nothing is scheduled or built.
sidebar:
  order: 18
---
**Status**: Proposed
**Date**: 2026-11-22
**Deciders**: Project maintainers
**Relates to**: [ADR-010](/ai-gateway/adrs/010-cost-attribution-pipeline/), [ADR-011](/ai-gateway/adrs/011-bedrock-guardrails-integration/), [ADR-014](/ai-gateway/adrs/014-two-plane-architecture-split/), [ADR-016](/ai-gateway/adrs/016-control-plane-api-foundation/), [ADR-017](/ai-gateway/adrs/017-agentgateway-data-plane-spike/)

This ADR records a direction. Nothing in it is scheduled or built.

## Decision (lead)

If the gateway ever keeps agent transcripts for eval harvesting and analytics, it does so through a governed lake behind a classification gate. Transcripts land in a short-retention quarantine zone, get classified against a tenant-owned sensitive-data taxonomy, and only then get promoted into a queryable store. Classification is the core of the feature. Entity-level PII redaction is a supporting piece.

## Context

Teams running coding assistants at scale want a transcript lake, but developers sometimes discuss sensitive material with the assistant. The hard case is not entity PII. HR, performance management, legal and other sensitive conversations contain no detectable entities, so no regex or PII classifier catches them. Deciding whether a conversation belongs in a lake at all is a data-classification problem.

What exists on main today:

- The data plane is the upstream agentgateway image pinned by digest ([ADR-017](/ai-gateway/adrs/017-agentgateway-data-plane-spike/)). Content safety runs inline through the Bedrock ApplyGuardrail API in detect and log-only mode, so the request path never rewrites content.
- `cost_attribution` parses agentgateway's flat access log, which carries identity and token counts and no bodies.
- `audit_pipeline` writes control-plane audit events and guardrail decisions through Firehose into Apache Iceberg tables on S3 Tables (ADR-016, ADR-017). `audit_query` reads them with Athena. The older `audit_log` module (Firehose to Parquet with a Glue catalog) is the legacy pipeline that `audit_pipeline` succeeds.
- Nothing captures request or response bodies, and there is no classifier, quarantine bucket or transcript store.

## Core positions

1. **The privacy boundary is at rest, before promotion, never in the request path.** The data plane stays fast and lossless, and nothing becomes queryable until the gate has run. In-path guardrails stay detect-only, as ADR-017 sets.
2. **Classification comes first, PII second.** The gate answers "should this conversation exist in the lake", not only "which tokens look like PII".
3. **Three actions: drop, restrict, redact.** The default taxonomy is small (HR, legal and corp-dev drop; security incidents and regulated markings restrict; entity PII redacts) and tenants extend it. Drop keeps a metadata stub only: team, timestamps, token counts and a category label, never the justification span.
4. **Shadow mode before any gating.** Nobody knows what share of traffic is sensitive, so any threshold picked up front is a guess. Measure first, then decide where the drop line goes.
5. **No human review queue for borderline HR content.** A reviewer reading a possibly sensitive HR transcript is the harm the feature exists to prevent. The gate over-drops instead, because a lost eval trajectory costs little and a leaked HR conversation costs a lot.

## Phase 1 is shadow-only

Phase 1 is body capture, quarantine, classification and a manifest of labels and confidences, plus a measurement report (share of traffic per category and the confidence distribution). It promotes nothing, builds no lake and redacts nothing. Everything after Phase 1 waits for that report.

## Provisional choices

These are the current leaning, to be confirmed with shadow data, and the positions above don't depend on them:

- Promoted records use the ATIF v1.7 trajectory format.
- The promoted store is Iceberg on S3 Tables, matching `audit_pipeline`.
- Access to restricted partitions uses Lake Formation grants.

## Alternatives considered

- **In-path redaction at the gateway.** Rejected. Coding traffic is full of PII lookalikes (emails in git logs, account IDs in ARNs), and a false positive in flight breaks the product.
- **PII-only scope.** Rejected. It fails the motivating case, because sensitive HR or legal content has no detectable entities.
- **Redact instead of drop for topic-sensitive content.** Rejected. The topic is the sensitive datum, and masking entities doesn't make the conversation safe to store.
- **A separate plain S3 and Glue Parquet store.** Rejected. `audit_pipeline` already puts Iceberg on S3 Tables on main, which gives managed compaction and snapshots, so a Parquet and Glue store would add a second pattern for no gain.

## Consequences

Positive: a lake with a provable gate, where the manifest shows that classification ran without re-exposing what it removed. Eval harvesting gets a standard format. The measurement report turns a vague worry into a number.

Negative: classifier recall is unproven until shadow data exists, which is why the design accepts over-dropping. Judge cost grows with developer count and must be modeled in Phase 1. Body capture adds data-plane log volume. Code-aware ambiguity (an email in `git blame` output the developer asked about) needs a tenant policy, defaulting to redact.

## Open questions

- **How do request and response bodies leave agentgateway v1.7.0?** Nothing on main captures bodies, and ADR-017 pins the upstream image with no custom build. The agentgateway docs read on 2026-11-22 (standalone "latest", not tied to v1.7.0) describe three candidates: a request-log database where `frontendPolicies.accessLog.database.llm: full` stores prompt and completion payloads, OpenTelemetry export of logs and traces described as including request and response bodies, and CEL access-log fields that can read request data. Whether v1.7.0 supports any of these, and whether the rendered config can use them on ECS, is unestablished. Check against the v1.7.0 config schema before Phase 1.
- Where quarantine lives and how it joins the access log that `cost_attribution` reads.
- Which taxonomy categories ship by default, and who owns the tenant policy.
- Whether secrets in transcripts, entity redaction detail and identity pseudonymization need their own ADRs.

## Sources

- agentgateway, View and customize logs (read 2026-11-22): https://agentgateway.dev/docs/standalone/latest/documentation/observability/access-logs/view/
- agentgateway, Observe traffic (read 2026-11-22): https://agentgateway.dev/docs/standalone/latest/documentation/llm/observability/
- Harbor ATIF RFC 0001: https://github.com/harbor-framework/harbor/blob/main/rfcs/0001-trajectory-format.md
- Amazon S3 Tables: https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html
