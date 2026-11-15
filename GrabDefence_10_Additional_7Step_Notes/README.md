# Grab Round 2 — 10 Additional Seven-Step System Design Guides

These are **additional** topics to complement the earlier 12 general design questions and the three GrabDefence screening guides. Every guide includes a seven-step interview walkthrough, Mermaid architecture, illustrative HTTP APIs, PostgreSQL data model, correctness considerations, failures, testing, monitoring, deployment and trade-offs.

> All numerical assumptions and API/database samples are **for interview practice**, not representations of Grab or GXBank production systems.

## Contents

1. [Account Takeover (ATO) Prevention](01_Account_Takeover_Prevention.md) — Digital Banking · Security
2. [Bank Transfer Authorization and Daily Limits](02_Bank_Transfer_Limits_Authorization.md) — Digital Banking · Concurrency
3. [Digital Bank Customer Onboarding / eKYC](03_Digital_Bank_Onboarding_eKYC.md) — Digital Banking · Asynchronous Workflow
4. [Bank Reconciliation and Settlement](04_Bank_Reconciliation_Settlement.md) — Fintech · Data Correctness
5. [Fraud Investigation / Case Management](05_Fraud_Investigation_Case_Management.md) — Digital Banking · Workflow and Audit
6. [Distributed Task Scheduler](06_Distributed_Task_Scheduler.md) — General Backend · Distributed Coordination
7. [Search and Autocomplete Service](07_Search_Autocomplete_Service.md) — General Backend · Read-Heavy Indexing
8. [API Gateway and Authentication Platform](08_API_Gateway_Authentication_Platform.md) — General Backend · Networking and Security
9. [Centralized Logging, Metrics and Tracing Platform](09_Centralized_Logging_Metrics.md) — General Backend · Observability
10. [Feature Flags and Configuration Service](10_Feature_Flag_Configuration_Service.md) — General Backend · Extensibility and Safe Rollouts

## Suggested short preparation sequence

1. **Transfer Limits** — bank transaction concurrency, state transitions and reconciliation.
2. **Account Takeover** — multi-factor security, device trust and safe degraded behavior.
3. **Distributed Scheduler** — general distributed systems and crash recovery.
4. **Onboarding and Case Management** — asynchronous workflows and auditability.
5. **Search and Feature Flags** — unfamiliar, read-heavy/extensible architecture patterns.
6. Skim the remaining two: **Reconciliation** and **Gateway**; **Observability** is useful for every mock.

## How to rehearse

Spend 5 minutes clarifying, 5 on requirements, 3 on scale, 10 on architecture, 10 on APIs and schema, 10 on failures/concurrency, then 5–10 on testing/monitoring/deployment/trade-offs. Use a whiteboard and answer the follow-up questions without notes.