# Serverless Runtime Documentation Review

**Review Date:** 2026-01-27
**Reviewer:** Claude Code
**Documents Reviewed:**
- `1_PRD.md` - Product Requirements Document
- `2_ADR_FUNCTION_ENTRYPOINT.md` - Architecture Decision Record for Function Entrypoints
- `3_ADR_STARLARK_RUNTIME.md` - Architecture Decision Record for Starlark Runtime

## Executive Summary

The documentation provides a comprehensive foundation for the Serverless Runtime module with strong business requirements and technical architecture. However, there are notable gaps in cross-document consistency, missing implementation details for several P0 requirements, and incomplete API specifications.

**Overall Assessment:**
- **PRD Completeness:** 95% - Excellent business requirements coverage
- **ADR Completeness:** 70% - Good technical foundation but missing key details
- **Consistency:** 75% - Some terminology and cross-reference issues
- **Actionability:** 65% - Missing several implementation specifications

---

## Critical Issues (P0)

### 1. Missing Core API Definitions

**Impact:** Cannot implement complete system without these APIs

The entrypoint ADR defines invocation APIs but omits several critical management APIs:

- **Function/Workflow Registration API** - How do users register new functions/workflows?
  - POST endpoint for registration
  - Validation and error responses
  - Versioning semantics (BR-019)
- **Function/Workflow Update API** - How are definitions updated?
  - Update vs. replace semantics
  - Impact on in-flight executions (BR-030)
- **Schedule Management API** - Required by BR-023, BR-176
  - Create/update/pause/resume/delete schedules
  - Missed schedule policy configuration
- **Tenant Enablement API** - Required by BR-021
  - Tenant provisioning for serverless runtime capability
  - Quota and governance setup

**Recommendation:** Add a new section "Management APIs" to `2_ADR_FUNCTION_ENTRYPOINT.md` covering these endpoints.

---

### 2. Security Context and Authentication Not Addressed

**Impact:** Cannot implement secure multi-tenant execution

Multiple PRD requirements reference security context but ADRs lack detail:

- **BR-006:** Execution identity context (system/API client/user)
- **BR-014:** Credential refresh for long-running workflows
- **BR-025:** Security context availability throughout execution
- **BR-086:** Security context propagation patterns

**Missing Details:**
- How is the security context passed to the runtime?
- How does `ctx` in `main(ctx, input)` expose security identity?
- What happens when a token expires during a multi-day workflow?
- How are credentials isolated in host-worker mode?

**Recommendation:** Add a "Security Context" section to `3_ADR_STARLARK_RUNTIME.md` defining:
- `ctx.identity` structure
- Token refresh mechanism
- Credential access patterns via `r_*` helpers

---

### 3. Tenant Isolation Architecture Undefined

**Impact:** Multi-tenant deployment architecture unclear

The PRD emphasizes tenant isolation (BR-002, BR-021, BR-169, BR-170) but ADRs don't specify:

- How are function/workflow definitions isolated per tenant?
- Are there separate registries, or filtered views of a shared registry?
- How does the runtime enforce tenant boundaries at execution time?
- What prevents cross-tenant access to execution state or secrets?

**Recommendation:** Add a "Tenant Isolation" section to `2_ADR_FUNCTION_ENTRYPOINT.md` covering:
- Registry architecture (per-tenant namespaces in identifiers?)
- Runtime enforcement mechanisms
- Data isolation guarantees

---

### 4. Incomplete Audit and Observability Specification

**Impact:** Cannot meet compliance requirements (BR-024, BR-035)

The PRD mandates comprehensive audit trails but ADRs lack:

- **What events are audited?**
  - Definition creation/modification/deletion
  - Execution lifecycle events
  - Permission changes
- **Audit record schema**
- **How are audit logs accessed?** (API endpoints)
- **Retention and immutability guarantees**

**Missing from Executions API:**
- Log streaming endpoint for live execution logs
- Metrics collection details (what metrics, how often?)
- Alert configuration (BR-109)

**Recommendation:** Add "Audit API" and "Logging API" sections to `2_ADR_FUNCTION_ENTRYPOINT.md`.

---

### 5. Streaming Execution Not Defined

**Impact:** Cannot implement streaming use cases mentioned in PRD

Entrypoint ADR mentions streaming (lines 90-92) but provides no details:

- What does streaming invocation look like? (WebSocket, SSE, gRPC streams?)
- How are durable streams implemented?
- How does checkpoint/resume work with streams?
- What is the API surface?

**Recommendation:** Either remove streaming references or add a complete "Streaming API" section to `2_ADR_FUNCTION_ENTRYPOINT.md`.

---

### 6. Hot Module Loading Not Specified

**Impact:** Cannot implement BR-136 (hot module reloading)

The PRD requires hot module loading (BR-136) but ADRs don't address:

- How are native modules registered at runtime?
- How do native modules integrate with the common calling mechanism?
- What is the API surface for native module functions?
- How are native modules versioned?

**Recommendation:** Add "Native Module Integration" section to `2_ADR_FUNCTION_ENTRYPOINT.md` or create a new ADR for this feature.

---

### 7. Compensation Failure Handling Undefined

**Impact:** Cannot reliably implement saga pattern (BR-033)

Starlark ADR describes compensation invocation but doesn't specify:

- What happens if a compensation function fails?
- Are compensations retried? With what policy?
- What if multiple compensations fail?
- How are compensation failures surfaced to operators?

**Recommendation:** Expand the "Workflow orchestration hooks" section in `3_ADR_STARLARK_RUNTIME.md` with:
- Compensation error handling semantics
- Retry policy for compensations
- Dead letter handling for failed compensations

---

## Important Issues (P1)

### 8. Static Traits for Dynamic Functions (BR-137)

**Impact:** Application developers cannot define reusable interfaces

PRD requirement BR-137 states: "The system MUST provide application developers a mechanism to define a reusable function interface that can be bound to a dynamic function at runtime."

This concept is not addressed in either ADR. It appears related to the GTS schema model but needs clarification:

- Is this about defining abstract function signatures that tenants implement?
- How does binding work?
- Is this similar to interfaces or traits in programming languages?

**Recommendation:** Clarify this requirement or add an ADR section explaining the design.

---

### 9. Validation Rules Incomplete in Starlark ADR

**Impact:** Cannot implement safe code validation

Starlark ADR mentions validation (lines 33-46) but doesn't specify:

- **Resource limits:**
  - Maximum code size
  - Maximum execution step budget
  - Maximum loop iterations
- **Forbidden constructs:**
  - Which built-ins are disabled?
  - Are dynamic imports (`load()`) allowed?
  - Are recursion limits enforced?
- **Policy enforcement examples**

**Recommendation:** Add a "Code Validation Rules" section to `3_ADR_STARLARK_RUNTIME.md` with:
- Complete list of allowed/disallowed built-ins
- Resource limit specifications
- Example validation failure scenarios

---

### 10. Type System Mapping Gaps

**Impact:** Runtime behavior for edge cases undefined

Starlark ADR mentions strong typing but doesn't define:

- **Optional fields:** How are JSON Schema optional properties handled in Starlark? (`None`?)
- **Enums:** How are JSON Schema enums represented in Starlark?
- **Nullable types:** What's the semantic difference between optional and nullable?
- **Type coercion:** What happens when Starlark code returns `1` but schema expects `"1"`?
- **Additional properties:** How does `additionalProperties: false` interact with Starlark dicts?

**Recommendation:** Add a "Type System Mapping" section to `3_ADR_STARLARK_RUNTIME.md` with:
- JSON Schema type → Starlark type mapping table
- Optional/nullable semantics
- Type coercion rules
- Validation error examples

---

### 11. Error Handling Inconsistencies

**Impact:** Custom error types unclear

Multiple inconsistencies around error handling:

1. **Custom errors:** Can functions define custom error types beyond the base runtime errors? The entrypoint schema has an `errors` field but it's unclear how custom errors are created and thrown.

2. **Error propagation:** If a function calls another function that fails, how is the error propagated? Is there implicit wrapping?

3. **Error serialization:** How are Starlark error objects serialized to the JSON error envelope?

**Recommendation:** Add an "Error Handling Guide" section to `3_ADR_STARLARK_RUNTIME.md` covering:
- How to define custom error types
- `r_exit(...)` usage with custom errors
- Error propagation semantics
- Complete error creation example

---

### 12. Missing Dead Letter Queue Specification

**Impact:** Cannot implement BR-028

PRD requires dead letter handling (BR-028) but no ADR addresses:

- When are executions moved to DLQ?
- What is the DLQ data model?
- How do operators access and recover DLQ items?
- Is there a DLQ API?

**Recommendation:** Add a "Dead Letter Queue" section to `2_ADR_FUNCTION_ENTRYPOINT.md`.

---

### 13. Incomplete Execution Record Model

**Impact:** Execution state unclear

The Jobs API returns execution state but several fields are ambiguous:

- **`client_id` vs `correlation_id`:** What's the semantic difference? When should each be used?
- **`job_id` vs `execution_id`:** The docs say they're the same for async, but when would they differ?
- **`observability.metrics`:** When are metrics populated? Only after completion?
- **Execution retention:** How long are execution records retained? (BR-107, BR-126)

**Recommendation:** Add an "Execution Data Model" section to `2_ADR_FUNCTION_ENTRYPOINT.md` clarifying these fields and their lifecycle.

---

### 14. Missing Examples

**Impact:** Harder for developers to understand end-to-end usage

The documents lack practical end-to-end examples:

- **Complete flow:** Registration → sync invocation → checking observability
- **Complete flow:** Registration → async invocation → status polling → result retrieval
- **Versioning example:** How to publish v2 of a function and migrate traffic (BR-019, BR-121)
- **Retry policy example:** How do retry policies interact with compensation?
- **Schedule example:** How to create a scheduled workflow (BR-023)

**Recommendation:** Add an "End-to-End Examples" appendix to `2_ADR_FUNCTION_ENTRYPOINT.md`.

---

## Consistency Issues

### 15. Terminology Inconsistencies

**Impact:** Confusion for readers

Multiple terms are used somewhat interchangeably:

| Concept | Terms Used | Preferred Term |
|---------|------------|----------------|
| A running function/workflow | "execution", "invocation", "job", "instance" | **Recommend:** "execution" (general), "job" (async execution record) |
| The serverless system | "runtime", "platform", "system" | **Recommend:** "Serverless Runtime" (system), "executor" (language runtime) |
| Function/workflow code | "implementation", "code", "definition", "program" | **Recommend:** "definition" (schema), "implementation" (code) |

**Recommendation:** Add a "Terminology" glossary section at the start of each ADR and use terms consistently.

---

### 16. Cross-Reference Gaps

**Impact:** Difficult to trace requirements to implementation

Cross-document references are inconsistent:

- Starlark ADR references entrypoint ADR by filename (line 17) but doesn't reference specific sections
- PRD requirements (BR-XXX) are not referenced in ADRs
- ADRs don't reference which PRD requirements they satisfy

**Recommendation:**
- Add a "Requirements Traceability Matrix" section to each ADR mapping PRD requirements to ADR sections
- Use explicit section references in cross-document links

---

### 17. Status Values Inconsistent Representation

**Impact:** API consumers need to handle multiple formats

The entrypoint ADR defines status as GTS IDs (e.g., `gts.x.core.faas.status.v1~x.core._.running.v1~`) but examples sometimes use short forms like "running".

**Clarify:**
- Should APIs accept/return short forms?
- Should there be a canonical mapping table?
- Are short forms client conveniences or server-supported?

**Recommendation:** Add a note clarifying that APIs MUST use full GTS IDs and short forms are documentation conveniences only.

---

## Completeness Gaps

### 18. Missing Decision Rationale

**Impact:** Cannot understand trade-offs or revisit decisions

Both ADRs lack:

- **Alternatives considered:** Why was JSON-RPC chosen over REST or gRPC?
- **Trade-offs:** What are the downsides of the chosen approach?
- **Future considerations:** What might change in v2?

**Recommendation:** Add "Alternatives Considered" and "Trade-offs" sections to each ADR per standard ADR format.

---

### 19. No Migration/Adoption Strategy

**Impact:** Unclear how to roll this out

Neither ADR addresses:

- How do existing workflows/jobs migrate to this system?
- Is there a compatibility layer or migration tool?
- What's the adoption path for teams currently using other orchestration systems?

**Recommendation:** Add an "Adoption Strategy" section to `2_ADR_FUNCTION_ENTRYPOINT.md`.

---

### 20. Host-Worker Mode Undefined

**Impact:** Cannot implement BR-207 (process isolation)

The PRD describes host-worker mode (BR-207) for stronger isolation but ADRs don't address:

- What is the host-worker communication protocol?
- How does the worker request runtime operations from the host?
- How are secrets segregated?
- What resources can workers access directly vs. through the host?

**Recommendation:** Add a "Host-Worker Isolation Architecture" section to `3_ADR_STARLARK_RUNTIME.md` or create a separate ADR.

---

### 21. Incomplete TODO Sections

**Impact:** Documents are explicitly incomplete

Line 1671 in `2_ADR_FUNCTION_ENTRYPOINT.md` has:
```markdown
## TODO:
```

This indicates unfinished work.

**Recommendation:** Either complete the TODO sections or remove them if no longer needed.

---

## Documentation Quality Issues

### 22. Schema Validation

**Impact:** JSON Schema examples may be invalid

Several JSON Schema examples use `"const"` with complex objects, which is valid JSON Schema but should be double-checked:

- Entrypoint ADR line 639-648: Using `const` with nested object
- Is `"const": {...}` the intended pattern or should it be `"properties": {...}` with `"additionalProperties": false`?

**Recommendation:** Validate all JSON Schema examples against the JSON Schema specification and common validators.

---

### 23. Missing Non-Functional Requirements in ADRs

**Impact:** Cannot implement performance targets

The PRD defines performance SLOs (BR-208, BR-209) but ADRs don't address:

- How will the runtime achieve p95 latency ≤ 100ms?
- What architectural choices support 10,000 concurrent executions?
- What's the persistence strategy for sub-200ms query latency?

**Recommendation:** Add a "Performance Architecture" section to an ADR or create a new "Performance and Scalability ADR".

---

### 24. Comparison Section Incomplete

**Impact:** Competitive analysis lacks depth

The "Comparison with similar solutions" section (lines 1673-1739 in entrypoint ADR) provides a good start but:

- Doesn't cover Temporal (mentioned in open-source landscape)
- Doesn't cover Azure Durable Functions in detail
- Doesn't highlight Hyperspot's unique advantages clearly

**Recommendation:** Expand the comparison section or move it to a separate competitive analysis document.

---

## Minor Issues

### 25. Formatting and Style

- Some code blocks use inconsistent indentation
- GTS ID examples could be more consistent (sometimes with `gts://` prefix, sometimes without)
- Table formatting could be improved for readability

### 26. Missing Diagrams

**Impact:** Complex flows harder to understand

Several sections would benefit from diagrams:

- Function/workflow execution lifecycle state machine
- Sync vs async invocation flow
- Workflow snapshot and resume flow
- Host-worker architecture (when defined)
- Compensation execution order diagram

**Recommendation:** Add architectural diagrams to ADRs using Mermaid or similar.

---

## Positive Aspects

Despite the issues identified, the documentation has many strengths:

✅ **Comprehensive PRD** - Excellent P0/P1/P2 prioritization and detailed requirements
✅ **Strong GTS-based typing** - Novel and well-thought-out approach to function contracts
✅ **Unified function/workflow model** - Clever design that avoids AWS's split products
✅ **JSON-RPC for sync** - Good choice for standardized invocation
✅ **Observability-first** - Execution timeline and debug APIs are well-designed
✅ **Clear error taxonomy** - Structured error types with proper inheritance
✅ **Good Starlark rationale** - Strong justification for language choice
✅ **Versioned runtime helpers** - `r_*_v1` pattern supports evolution

---

## Recommendations Summary

### Immediate Actions (Critical Path)

1. **Add Management APIs** to entrypoint ADR (registration, update, schedules, tenant enablement)
2. **Define Security Context** model in Starlark ADR
3. **Specify Tenant Isolation** architecture in entrypoint ADR
4. **Expand Audit/Observability** specifications
5. **Complete or remove TODO sections**

### Important Follow-ups

6. Define streaming execution or remove references
7. Clarify static traits (BR-137)
8. Add detailed validation rules to Starlark ADR
9. Define type system mapping comprehensively
10. Add end-to-end examples

### Quality Improvements

11. Add terminology glossary to both ADRs
12. Create requirements traceability matrix
13. Add architectural diagrams
14. Expand alternatives/trade-offs sections
15. Improve cross-document referencing

---

## Conclusion

The Serverless Runtime documentation provides a strong foundation with excellent business requirements and a thoughtful technical approach. The main gaps are in **implementation specifications for core management APIs**, **security context details**, and **comprehensive error handling**. Addressing the critical issues will make the design actionable for implementation.

**Estimated Effort to Address Critical Issues:** 3-5 days of focused documentation work

**Recommended Next Steps:**
1. Review this feedback with the team
2. Prioritize critical issues based on implementation timeline
3. Assign owners to each documentation gap
4. Update ADRs incrementally with reviews after each major addition
