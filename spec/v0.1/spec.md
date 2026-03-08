# Delegauth Open Specification v0.1 (Protocol Proposal)

**Authority defines autonomy.**  
**Actionable autonomy without explicit authority is incomplete.**

## 1. Introduction

Modern systems are increasingly **execution-capable**: they can initiate actions that change operational reality (e.g., issuing refunds, updating records, triggering workflows, committing transactions).

Once a system can act, it operates in the domain of **authority**. In enterprise contexts, authority cannot remain implicit (in code, prompts, or scattered configuration). It must be defined as a structured, reviewable mandate.

Delegauth specifies a portable **authority definition** for action-taking systems: scope, limits, human review, accountability, and minimal integration semantics.

## 2. Definitions

- **Actionable autonomy:** behavior that can initiate actions with real-world effect (beyond informational output).
- **Authority:** formally delegated decision power to execute actions within defined boundaries.
- **Mandate:** an approved instance of delegated authority represented as a Delegauth policy.
- **Responsible authority:** the named human role/person accountable for defining and approving a mandate.
- **Static limit:** per action/per case constraint (e.g., max refund per ticket).
- **Dynamic limit:** aggregated constraint across an observation window (business day).
- **Business day window:** time interval (timezone + local reset time) used to compute dynamic limits.
- **Overflow:** any limit exceedance or inability to evaluate limits with certainty.
- **Human review:** explicit approve/deny decision required for an overflowed action.
- **Integrity safeguard:** deterministic incident triggers that suspend autonomy (exception only).

## 3. Authority Model (Normative)

A Delegauth policy **MUST** include:
1) Identity  
2) Action scope (allowed + restricted)  
3) Financial limits (static)  
4) Dynamic limits (business day aggregation)  
5) Overflow handling rules  
6) Integrity safeguards  
7) Audit requirements  
8) State management

### 3.1 Identity (MUST)
- `entity_name` (string)
- `entity_type` (string/enum)
- `organization` (string)
- `responsible_authority` (string)
- `effective_date` (date)

### 3.2 Action Scope (MUST)
- `allowed_actions` (array)
- `restricted_actions` (array)

Rules:
- Any action not explicitly allowed **MUST** be denied.
- If an action is restricted, it **MUST** be denied (restricted wins).

### 3.3 Limits (MUST)
Static (per case):
- `max_refund_per_ticket`, `currency`

Dynamic (business day aggregation):
- `business_timezone` (IANA timezone string, e.g., `Europe/Zurich`)
- `reset_time_local` (`HH:MM`)
- `max_refunds_per_business_day`
- `max_total_refund_per_business_day`
Optional (recommended for abuse containment):
- per customer equivalents

### 3.4 Overflow Handling (Normative)
For v0.1, Delegauth uses deterministic overflow behavior:

- If any static or dynamic limit is exceeded, or cannot be evaluated with certainty, the triggering action **MUST NOT** execute automatically.
- The action **MUST** be routed to **human review** (approve/deny).
- The system enters a restricted mode conceptually equivalent to **PAUSED** where financial actions (refunds) are blocked until review.

Note: PAUSED is an internal enforcement state name; user-facing materials should describe it as “human review required / restricted mode”.

## 4. State & Stop Protocol (Normative)

Delegauth enforcement uses four states:
- `ACTIVE` — normal operation subject to scope/limits
- `PAUSED` — restricted mode (e.g., refunds blocked; operational actions may continue)
- `SUSPENDED` — incident mode; only escalation allowed
- `TERMINATED` — mandate revoked; no actions allowed

### 4.1 Limits => PAUSED (always)
Limit overflow **MUST** route to human review and restricted mode (`PAUSED`).  
v0.1 does **not** auto-suspend due to limit overflow.

### 4.2 Integrity Safeguards => SUSPENDED (exception only)
`SUSPENDED` is reserved for deterministic integrity/incident triggers (no ML), e.g.:
- audit write failures when audit is mandatory
- repeated inability to evaluate limits
- restricted action attempt
- policy mismatch/stale policy
- duplicate execution risk

In `SUSPENDED`, all actions except escalation **MUST** be denied until explicit human reactivation.

## 5. Runtime Interface (Decision Contract)

Delegauth specifies a minimal runtime contract between a caller (agent/workflow) and an enforcement layer.

### 5.1 AuthorizationRequest (MUST)
A request **MUST** include:
- agent_id, policy_id, policy_version
- action, reference {type, id}
- parameters (action-specific), timestamp_utc

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

Semantics:
- `PAUSE` means “human review required”; the action must not execute without explicit approval.
- Approvals must be idempotent by `decision_id` (execute at most once).

## 6. Audit Requirements (Minimum)

If audit requirements are enabled (recommended for v0.1), enforcement **MUST** record:
- action attempts and outcomes
- state transitions
- overflow events
- human approve/deny decisions

Audit events should include: timestamp, mandate_id, policy_hash, action, decision, reason_code, state_before/after, reference_id, actor (if human).

## 7. Non-Goals

Delegauth is not:
- a monitoring/observability product
- an anomaly detection or ML safety system
- an IAM replacement (authentication/identity)
- a runtime orchestration engine
- a compliance platform
- a blockchain / distributed ledger component

Delegauth defines authority artifacts and requirements; enforcement is implemented by the hosting system.

## 8. Versioning

- v0.1: Refund authority mandate pack + minimal decision contract.
- Breaking changes require a version bump.
