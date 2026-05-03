---
layout:     post
title:      "Sample data"
description: "Treat sample data as code"
date:    2026-05-03
author: "Paul"
publishDate: 2026-05-03
tags:
  - data
  - Programming
  - Productivity
categories: [ code ]
---

## The Strategic Case for Programmable Sample Data

### I. The Core Thesis
Manual database entry is a bottleneck that introduces inconsistency. The most effective way to ensure environmental parity and development velocity is to treat **sample data as code**. By leveraging business services to seed the database with deterministic, UUID-based records, teams create a stable foundation for testing, UI development, and collaborative debugging.

---

### II. Supporting Arguments

#### 1. Enforcement of Business Logic via Service Seeding
A common mistake is seeding a database via direct SQL scripts. This bypasses the application’s validation rules and side effects.
* **Logic Parity:** Using your actual business services to generate data ensures the database respects the same constraints as your production environment.
* **Regression Testing:** If a seed script fails due to a logic change in a service, you have found a breaking change before it ever reaches a staging environment.

#### 2. UI Stability and Deterministic State
Frontend development requires a predictable data shape to build robust components.
* **Consistent Mocking:** Sample data provides the UI with a reliable set of entities, preventing the "empty state" hurdle during early-stage development.
* **Deterministic IDs:** By using hardcoded UUIDs in your seed scripts rather than relying on database-generated sequences, you ensure that specific URLs and resource paths remain constant. This allows team members to share bookmarks or deep links to specific records during the dev cycle.

#### 3. Rapid Iteration and Environment Recovery
The ability to "nuke" and rebuild a local environment is essential for maintaining a clean development state.
* **Schema Evolution:** When modifying table structures (e.g., via Exposed or similar DSLs), the ability to immediately repopulate the new schema with valid data reduces downtime.
* **Exploration:** Developers can safely experiment with destructive actions knowing they are one command away from a fresh, fully-populated state.

---

### III. Technical Implementation Guidelines

#### Strategy: The "Seeder" Service
Instead of raw SQL, implement a dedicated `SeedService`. This service coordinates the creation of entities in the correct order to satisfy foreign key constraints.

| Feature | Implementation Detail | Benefit |
| :--- | :--- | :--- |
| **Identity** | Hardcoded `UUID.fromString()` | Bookmarkable URLs across local/dev environments. |
| **Logic** | Call `AccountService.create()` | Exercises interceptors, audit logs, and validation. |
| **Lifecycle** | Hook into App Startup / CLI | Ensures the database is ready immediately after migration. |

#### Handling Relationships
When creating sample data, model "Personas" (e.g., `ADMIN_USER_ID`, `STANDARD_USER_ID`). This allows you to test different permission levels and data visibility rules (like those managed via Permify) consistently.

#### Avoiding "Data Rot"
Treat your seed scripts with the same rigor as your production code. Review them during PRs to ensure that as your business rules evolve, your "ideal" development state evolves with them.

---

> **A Note on Scalability:** While this approach adds a small amount of overhead to the initial setup, the ROI is realized every time a developer avoids a "reproducibility" bug or a broken UI state. It transforms the database from a black box into a predictable tool.
