---
title: "Evaluating bank-core v1-basic: Locking, Concurrency, and the Limits of Web-Prompted AI"
date: 2026-05-24T18:36:21+10:00
publishDate: 2026-05-24
draft: false
layout: "post"
tags: ["springboot", "concurrency", "database-locking", "ai-assisted-coding"]
categories: ["coding"]
description: "An engineering review of bank-core v1-basic: its concurrent locking strategies, O(1) database audits, browser-prompted AI flaws, and the transition to v2 spec-driven engineering."
author: "Paul"
---

`bank-core` version `v1-basic` successfully demonstrates deadlock-free pessimistic database locking and high-throughput $O(1)$ ledger validation, but suffers from anemic procedural layers typical of raw browser-prompted AI code. To fix these leaky boundaries, the upcoming `v2` will pivot to spec-driven development utilizing **Claude Code** to enforce domain-driven isolation.

> https://github.com/prule/bank-example/tree/v1-basic

---

## The Core Wins: OpenAPI, Locking, and DB Auditing

The primary goal of `v1-basic` was to refresh my memory with respect to java and spring boot after being in the Kotlin/Ktor ecosystem for a while.

The browser based prompting worked quite well for a prototype considering I didn't really know what I wanted to build - doing it this way allowed for very flexible discovery and prototyping. Obviously I could have done the same prompting in Claude Code or Antigravity from the beginning, but I chose the web way as the most basic option to compare with subsequent implementations.

From the prototype code I would go on to generate requirements and specs which I would later use with OpenSpec to compare outcomes.

Some of the attributes of this experiment:

### 1. OpenAPI Contract-First Generation

The public API contract is defined explicitly in `src/main/resources/static/openapi/`. During the Gradle build lifecycle, the `openapi-generator` plugin automatically scaffolds the underlying Spring MVC interface (`TransfersApi`) and all core request/response DTOs (`TransferRequest`). 

By enforcing this contract-first pattern, the network interface is decoupled from internal model definitions. The API contract remains the single source of truth, eliminating the risk of drifting controller signatures or out-of-sync parameter validations.

### 2. Deadlock-Free Pessimistic Locking

Cyclic deadlocks occur when concurrent threads request locks on hot accounts in different orders (e.g., Thread A locks Account 1 &rarr; 2, while Thread B locks Account 2 &rarr; 1). `v1-basic` eliminates this by sorting account identifiers lexicographically before acquiring row-level database locks (`SELECT FOR UPDATE`).

Concurrent operations queue up in a deterministic order, completely bypassing cyclic waiting states.

### 3. O(1) Ledger Validation

Rather than pulling transaction histories into application memory to verify that debits equal credits, the system offloads validation directly to the database layer using a single optimized native query:

```sql
SELECT CASE 
    WHEN SUM(CASE WHEN type = 'CREDIT' THEN amount ELSE -amount END) = 0 THEN 1 
    ELSE 0 
END
FROM ledger_transactions
WHERE journal_entry_id = :journalId
```

This prevents resource exhaustion at the JVM layer and keeps verification speed independent of transaction depth.

{{< notice type="tip" >}}
Moving math validations to the SQL layer guarantees atomic, instant audit checks while shielding the application from object-relational mapping (ORM) proxy inflation.
{{< /notice >}}

---

## Operational Safety: The Schedulers

To provide defense-in-depth, two background `@Scheduled` workers continuously reconcile the ledger and detect balances out-of-sync:

1. **`LedgerReconciliationScheduler` (10s Cadence):** Validates optimistic `PENDING` journals and promotes them to checked states.
2. **`BalanceDriftDetectorScheduler` (30s Cadence):** Audit-sweeps account snapshots against immutable ledger lines using a deterministic checkpoint range lock (`SystemConfig.lastBalanceCheckId`). If any snapshot drifts from the ledger sum, the account is immediately suspended.

---

## Where v1-basic Falls Short: The Browser AI Trap

Because `v1-basic` was built using raw, context-free chat sessions in the browser (using Google Gemini web interface), the codebase exhibits structural compromises:

* **Procedural Leakage:** Code reflects standard spring layered conventions. Business rules are scattered between controllers, services, and semi-rich models rather than maintaining a pure, isolated domain.
* **Tight Spring/Hibernate Coupling:** By skipping Ports & Adapters (Hexagonal Architecture), persistence details leak directly into core application logic, making unit testing difficult without starting full spring context slices.
* **Anemic Tendencies:** The lack of strict boundary separation matches the standard, highly procedural boilerplate that standard LLMs generate when prompted linearly in a browser tab without explicit tooling.

---

## The Roadmap: Pivot to v2 & Claude Code

To transition this procedural codebase into a highly modular, testable system, the next phase will focus on:

1. **Making the Domain Explicit:** Redefining core boundaries in accordance with clean domain-driven architecture to achieve local reasoning.
2. **Spec-Driven Development:** Defining rigid contracts upfront before generating or refactoring execution paths.
3. **Agentic Code Engineering:** Comparing the raw browser prompting of `v1` against structured agentic tools like **Claude Code** operating with customized developer harnesses and constraints. This will provide a side-by-side comparison of AI software engineering styles.

> In progress here: https://github.com/prule/bank-example/tree/v2-openspec-claude
