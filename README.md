# delegauth-spec
Open Specification v0.1 (Protocol Proposal) for delegated authority in action-taking systems.
# Delegauth Specification (v0.1) — Protocol Proposal

**Delegauth** is an **open specification proposal** for defining **delegated authority** for **action-taking systems** in enterprise contexts.

It standardizes a portable **mandate** that answers:
- **What** actions are allowed / forbidden
- **Which limits** apply (v0.1: financial + business-day aggregation)
- **How overflow is handled** (deterministic, human review)
- **When to treat behavior as an incident** (integrity triggers)
- **How to integrate** via a minimal runtime decision contract (ALLOW / PAUSE / DENY)

> Brand principle: **Authority defines autonomy.**  
> Core thesis: **Actionable autonomy without explicit authority is incomplete.**

## Status
- **v0.1** is a first public proposal focused on the MVP use case: **Customer Service Refund Authority**.
- This repository contains the **spec**, the **JSON schema**, and **example artifacts**.

## Contents
- **Spec:** `spec/v0.1/spec.md`
- **Schema:** `schema/delegauth-v0.1.schema.json`
- **Examples:** `examples/`

## Quick links
- Canonical spec (crawlable): https://holdhodor.github.io/delegauth-spec/
- Spec v0.1 (Markdown): https://holdhodor.github.io/delegauth-spec/spec/v0.1/spec.md
- Website / reference implementation: https://delegauth.com

## License
MIT (spec + schema + examples).
