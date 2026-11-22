# PRD-001: TraceTriage v1 — Product Requirements Document

**Linked BRD:** `docs/01_BRD.md`
**Date:** 2026-11-20
**Status:** Locked for v1 Design
**Target Corpus (v1):** 2,000–3,000 real resolved FastAPI GitHub issues + official FastAPI Markdown documentation

---

## 1. v1 Scope & Target User

**Target User:** Backend developers and on-call support engineers diagnosing FastAPI bugs, runtime exceptions, and configuration errors.

**Core v1 Workflow:**
A user submits a bug report (with optional stack trace, code snippet, and library versions) to `POST /v1/triage`. The system runs a bounded, deterministic retrieval-and-verification pipeline over indexed FastAPI docs and resolved GitHub issues:
1. Executes hybrid retrieval (lexical + dense vector) fused via Reciprocal Rank Fusion (RRF) and reranked by a cross-encoder.
2. Evaluates top evidence confidence; if borderline, executes **at most 1** deterministic query refinement retry.
3. Passes the abstention gate:
   * **Above threshold:** Packs deduplicated chunks within a hard token cap and generates a strictly cited root-cause brief (`RESOLVED`).
   * **Below threshold:** Blocks LLM root-cause guessing and returns a structured escalation brief (`NEEDS_HUMAN_TRIAGE`).

---

## 2. Input Contract (`POST /v1/triage`)

| Field | Type | Required | Validation Guardrail |
| :--- | :--- | :--- | :--- |
| `error_description` | `string` | Yes | Min 15 chars, max 4,000 chars (trimmed). Rejects empty or vague 1-word inputs like `"help"`. |
| `stack_trace` | `string \| null` | No | Max 12,000 chars. |
| `code_snippet` | `string \| null` | No | Max 4,000 chars. |
| `fastapi_version` | `string \| null` | No | Optional semver string (e.g., `"0.115.0"`), max 32 chars. |
| `pydantic_version` | `string \| null` | No | Optional semver string (e.g., `"2.9.2"`), max 32 chars. |

**Invalid Input Policy:**
* Oversized or under-specified inputs return **`HTTP 422 Unprocessable Entity`** with a clear field-level validation error.
* **No silent truncation on `stack_trace`:** Python tracebacks put the actual exception type and root message at the very *bottom*. Silently cutting off a 50,000-character log risks deleting the exact error signal without warning the caller.

---

## 3. Output Contract & Decision States

v1 uses a **strict binary decision state** (`RESOLVED` vs. `NEEDS_HUMAN_TRIAGE`). A third `"PARTIAL"` state is intentionally excluded in v1 so abstention accuracy can be measured unambiguously without hiding weak predictions in a middle bucket.

### Case A: `status = "RESOLVED"`

Returned only when retrieved evidence passes the calibrated reranker confidence gate and output passes programmatic citation validation.

```json
{
  "request_id": "uuid-v4",
  "status": "RESOLVED",
  "root_cause_summary": "Concise explanation of why the error occurs.",
  "recommended_fix": "Actionable fix grounded strictly in retrieved sources.",
  "citations": [
    {
      "source_id": "gh-issue-10421",
      "source_type": "github_issue",
      "url": "[https://github.com/fastapi/fastapi/issues/10421](https://github.com/fastapi/fastapi/issues/10421)",
      "relevant_excerpt": "Short verbatim snippet supporting the claim."
    }
  ],
  "diagnostics": {
    "top_rerank_score": 0.0,
    "retry_triggered": false,
    "context_tokens_used": 0,
    "latency_ms": 0
  }
}
```

### Case B: `status = "NEEDS_HUMAN_TRIAGE"`

Returned whenever top evidence remains below the confidence threshold after at most 1 retry, or when citation verification fails.

```json
{
  "request_id": "uuid-v4",
  "status": "NEEDS_HUMAN_TRIAGE",
  "abstention_reason": "Retrieved evidence below confidence threshold (no verified match in FastAPI corpus).",
  "queries_attempted": ["raw error query", "refined exception signature"],
  "unverified_leads": [
    {
      "source_id": "gh-issue-8812",
      "url": "[https://github.com/fastapi/fastapi/issues/8812](https://github.com/fastapi/fastapi/issues/8812)",
      "rerank_score": 0.0
    }
  ],
  "missing_context_hints": [
    "Specify fastapi_version and pydantic_version to rule out v1/v2 breaking changes."
  ],
  "diagnostics": {
    "top_rerank_score": 0.0,
    "retry_triggered": true,
    "context_tokens_used": 0,
    "latency_ms": 0
  }
}
```

### Hard Citation Integrity Rule

Every source_id returned inside citations is verified programmatically in Python against the exact set of chunk IDs retrieved for that request. If the generator fabricates an unretrieved ID, the response is rejected and downgraded to NEEDS_HUMAN_TRIAGE.

## 4. Engineering Budgets & Target SLAs (Design Guardrails)

Note: These are design ceilings to engineer against, not measured benchmark results.

| Constraint | v1 Target Ceiling | Engineering Rationale |
| :--- | :--- | :--- |
| Max retrieval retries | 1 (max 2 search passes total) | Prevents runaway loops and bounds worst-case latency. |
| Max LLM context budget | 3,000 tokens (retrieved chunks only) | Caps per-query cost, enforces deduplication, and prevents long-context degradation. |
| End-to-end p95 latency | < 5,000 ms (local/single-node target) | Keeps interactive triage usable while allowing local reranking + 1 LLM call. |
| Abstention confidence threshold | TBD (calibrated empirically) | Will be set using the 75-case offline eval set—never guessed in advance. |

## 5. Anticipated Failure Modes (Design Inputs for TDD)

### Exact Symbol & Stack Trace Blindness

Dense embeddings frequently miss exact HTTP status codes, exception names (`ResponseValidationError`, `pydantic_core.ValidationError`), or obscure parameter names. Addressed in TDD via PostgreSQL lexical search (`tsvector`) + dense vectors (`pgvector`) + RRF.

### Version Drift (Pydantic v1 vs. v2 / FastAPI Upgrades)

Retrieving an old 2021 GitHub issue recommending Pydantic v1 syntax (`.dict()`, `@validator`) for a developer running FastAPI with Pydantic v2 (`.model_dump()`, `@field_validator`) produces broken fixes. Addressed in TDD via version metadata preservation and version-conflict test cases.

### Out-of-Domain & Unreleased Bug Hallucinations

Users submitting Django/Flask traces or genuinely novel bugs not present in the corpus will trigger confident hallucinations in naive RAG pipelines. Addressed via the reranker confidence abstention gate and explicit trap queries in evals.

### Fabricated Citations

LLMs routinely invent plausible-looking issue numbers (e.g., `#9412`). Addressed via deterministic post-generation ID verification against the retrieved context pool.

## 6. v1 Acceptance Criteria (Offline Eval Gate)

v1 is considered complete only when the automated evaluation harness runs reproducibly over a fixed golden set of 50–60 real resolved FastAPI issues + 15 unanswerable trap queries and produces a logged comparison table across:

- Lexical Only (`tsvector`)
- Vector Only (`pgvector`)
- Hybrid (RRF)
- Hybrid + Cross-Encoder Reranker
- Full Pipeline + Abstention Gate

Required logged metrics (recorded honestly, win or lose): Recall@5, MRR@10, Programmatic Citation Validity (%), Abstention Accuracy on Trap Queries, False-Abstention Rate on Valid Bugs, p95 Latency (ms), and Avg Context Tokens / Query.

## 7. Explicit v1 Non-Goals (Deferred to v2)

- **Classical ML Complexity Router & Calibrated Model Tiering:** Deferred to v2—requires real logged v1 query distributions and failure cases before training a meaningful classifier.
- **Semantic Cache (Duplicate Bug Fast-Path):** Deferred to v2—pure latency/cost optimization added only after core retrieval and abstention are verified.
- **Open-Ended Multi-Tool Autonomous Agent / SQL Metadata Tooling:** Deferred to v2—v1 enforces a strict deterministic 2-step state machine.
- **Web UI, Auth/RBAC, or Automated GitHub Commenting/PR Creation:** Out of scope for v1.git add docs/02_PRD.md