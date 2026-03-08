# Delegauth Open Specification v0.1 (Protocol Proposal)

**Authority defines autonomy.**  
**Actionable autonomy without explicit authority is incomplete.**

## 1. Introduction

Modern systems are increasingly **execution-capable**: they can initiate actions that change operational reality (e.g., issuing refunds, updating records, triggering workflows, committing transactions).

Once a system can act, it operates in the domain of **authority**. In enterprise contexts, authority cannot remain implicit (in code, prompts, or scattered configuration). It must be defined as a structured, reviewable mandate.

Delegauth specifies a portable authority definition: scope, limits, human review, accountability, and minimal integration semantics.

## 2. Definitions

- **Actionable autonomy:** behavior that can initiate actions with real-world effect.
- **Authority:** formally delegated decision power to execute actions within defined boundaries.
- **Mandate:** an approved instance of delegated authority represented as a Delegauth policy.
- **Responsible authority:** named human role/person accountable for defining/approving a mandate.
- **Static limit:** per action/per case constraint (v0.1: refund-per-ticket).
- **Dynamic limit:** aggregated constraint across a business-day window.
- **Overflow:** limit exceedance or inability to evaluate limits with certainty.
- **Human review:** explicit approve/deny required for an overflowed action.
- **Integrity safeguard:** deterministic incident triggers that suspend autonomy (exception only).

## 3. Authority Model (Normative)

A Delegauth policy **MUST** include:
1) Identity  
2) Action scope (allowed + restricted)  
3) Limits (v0.1 scope defined below)  
4) Overflow handling rules  
5) Integrity safeguards  
6) Audit requirements  
7) State management

### 3.1 Identity (MUST)
- `entity_name`, `entity_type`, `organization`, `responsible_authority`, `effective_date`

### 3.2 Action Scope (MUST)
- `allowed_actions` (array)
- `restricted_actions` (array)

Rules:
- Any action not explicitly allowed **MUST** be denied.
- If an action is restricted, it **MUST** be denied (restricted wins).

### 3.3 Limits (v0.1 scope; Normative)

**Important:** v0.1 is intentionally scoped to the **Refund Authority** MVP.  
Therefore, v0.1 defines **financial limits + business-day aggregation**.  
Non-financial limits (e.g., operational quotas, data access constraints) are out of scope for v0.1 and may be generalized in a future version (e.g., `static_limits`).

**Static (financial) limits (MUST in v0.1):**
- `financial_limits.max_refund_per_ticket`
- `financial_limits.currency`

**Dynamic limits (business day aggregation) (MUST in v0.1):**
- `dynamic_limits.business_timezone` (IANA timezone string, e.g., `Europe/Zurich`)
- `dynamic_limits.reset_time_local` (`HH:MM`)
- `dynamic_limits.max_refunds_per_business_day`
- `dynamic_limits.max_total_refund_per_business_day`

Optional (recommended for abuse containment):
- `dynamic_limits.max_refunds_per_customer_per_business_day`
- `dynamic_limits.max_total_refund_per_customer_per_business_day`

### 3.4 Overflow Handling (Normative)

If any static/dynamic limit is exceeded, or cannot be evaluated with certainty:
- the triggering action **MUST NOT** execute automatically
- the action **MUST** be routed to **human review** (approve/deny)
- enforcement enters restricted mode conceptually equivalent to `PAUSED` (financial actions blocked until review)

Note: `PAUSED` is an internal enforcement state name; user-facing materials should describe it as “human review required / restricted mode”.

### 3.5 Policy Hash (Normative)

`policy_hash` provides artifact integrity and binding between PDF/JSON/instructions.

**Computation (v0.1):**
1) Take the full policy JSON **excluding** the `policy_hash` field.
2) Serialize to **canonical JSON**:
   - UTF-8
   - object keys sorted lexicographically
   - no extra whitespace (separators `,` and `:`)
3) Compute SHA-256 over the canonical JSON bytes.
4) Set `policy_hash = "sha256:<64 lowercase hex chars>"`.

This makes `policy_hash` deterministic and reproducible across implementations.

## 4. State & Stop Protocol (Normative)

States:
- `ACTIVE`, `PAUSED`, `SUSPENDED`, `TERMINATED`

v0.1 semantics:
- **Limits overflow => PAUSED (always)** (no auto-suspend due to limits)
- **SUSPENDED is exception-only** for deterministic integrity triggers (no ML), e.g.:
  - audit write failures (when mandatory)
  - repeated inability to evaluate limits
  - restricted action attempt
  - policy mismatch/stale policy
  - duplicate execution risk

In `SUSPENDED`, only escalation is permitted until explicit human reactivation.

## 5. Runtime Interface (Decision Contract)

### 5.1 AuthorizationRequest (MUST)

A request **MUST** include:
- `agent_id`, `policy_id`, `policy_version`
- `action`
- `reference` { `type`, `id` }
- `timestamp_utc`
- `parameters` (defined below for v0.1 refund use case)

### 5.2 AuthorizationDecision (MUST)

A decision **MUST** include:
- `decision`: `ALLOW` | `DENY` | `PAUSE`
- `decision_id` (unique)
- `current_state`
- `reason.code`
- `required_next_action`: `NONE` | `ESCALATE_TO_HUMAN` | `REQUEST_HUMAN_REACTIVATION`

### 5.3 Reason Codes (minimum)
- `STATE_DISALLOWS_ACTION`
- `ACTION_NOT_ALLOWED`
- `ACTION_RESTRICTED`
- `STATIC_LIMIT_EXCEEDED`
- `DYNAMIC_LIMIT_EXCEEDED`
- `LIMIT_EVALUATION_UNCERTAIN`
- `MANDATE_REVOKED`
- `INTEGRITY_SUSPENSION_TRIGGERED`

### 5.4 Parameters Schema (v0.1 Refund Use Case)

For v0.1, `parameters` is defined for `issue_refund`:

`action = "issue_refund"`:
- `parameters.amount` (number, >0)
- `parameters.currency` (string, 3-letter)
- `parameters.reason_code` (string, optional)

Other actions in v0.1:
- `answer_support_ticket`: parameters optional (ticket id is in `reference`)
- `escalate_to_human`: parameters optional (e.g., reason/message)

Implementations MAY extend parameters, but should document extensions and provide examples.

Approvals must be idempotent by `decision_id` (execute at most once).

## 6. Audit Requirements (Minimum)

Enforcement should record:
- action attempts and outcomes
- state transitions
- overflow events
- human approve/deny decisions

Audit events should include: timestamp, mandate_id, policy_hash, action, decision, reason_code, state_before/after, reference_id, actor (if human).

## 7. Non-Goals

Delegauth is not monitoring/observability, not ML anomaly detection, not IAM replacement, not a runtime orchestration engine, not a compliance platform, and not a blockchain component.

## 8. Versioning

- v0.1: Refund authority mandate + decision contract + schema/examples.
- Breaking changes require a version bump.
