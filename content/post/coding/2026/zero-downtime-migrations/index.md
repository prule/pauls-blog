---
title: "Zero-Downtime Database Migrations"
date: 2026-05-22T11:00:00+10:00
publishDate: 2026-05-22
draft: true
layout: "post"
tags: ["Databases", "Migrations", "Deployment", "Architecture"]
categories: ["code"]
description: "Zero-downtime is a property of how you migrate, not which database you picked. Here's the playbook."
author: "Paul"
---

Zero-downtime is a property of the migration, not the database. Postgres, MySQL, SQL Server — none of them break downtime for you. They give you the primitives. Whether the deploy goes out without a blip depends on the *shape* of the migration: how many steps, what runs when, what's reversible, and what the application is doing at each step.

The playbook below is what I reach for whenever a schema change touches a live table.

### Expand-Contract Is The Default

Every non-trivial migration follows the same five steps:

1. **Expand** — add the new column, table, or index. Nullable. No constraints that existing writes can violate.
2. **Dual-write** — application writes to both old and new. Old reads still served from the old column.
3. **Backfill** — copy historical data from old to new in batches. Never one big `UPDATE`.
4. **Cut over** — switch reads to the new column. Old column still being written, in case of rollback.
5. **Contract** — stop writing to the old column. Drop it in a later release.

Each step ships independently. Each is reversible. No step requires the app and the database to change at the same instant.

This is slower than "alter the table and redeploy". That is the point. Speed of a single migration is not what you're optimising for. Uninterrupted service is.

### Never Combine Schema, Data, And Code In One Release

The cardinal rule. If you change the schema, change the data shape, *and* change the code that reads it in a single deploy, you've created a window where any of three things can break with no clean rollback path.

Split them. Schema change ships first, alone. Code that tolerates both shapes ships second. Data migration runs in the background. Code that depends on the new shape ships last. Each release is small, observable, and revertable.

The teams that get burned by migrations are almost always the teams that bundled them with feature work. The migration looks like one task; it should ship as four.

### Feature Flags Belong On The Read Path

During cutover, the read path is where risk concentrates. The write path can dual-write safely; reads have to choose one source of truth.

Put a feature flag around the read. Default to old. Flip to new for 1% of traffic. Compare. Flip to 10%. Compare. Then 100%. If something is wrong — bad backfill, missing index, subtle type coercion — you find out at 1%, not at 100%. The flag is also the rollback: flip it back, no deploy required.

{{< notice type="tip" >}}
The read-path flag should compare results from both sources during the rollout window. Shadow reads — query both, return old, log the diff — catch correctness bugs that a simple traffic split won't.
{{< /notice >}}

### Write The Rollback First

The rollback plan is the migration plan in reverse. If you cannot describe it in writing before you start, you are not ready to migrate.

Concretely, before step 1 ships, you should know:

- How do I undo each step?
- Which steps are *not* reversible? (Dropping data, for one.)
- What's my detection signal — what tells me to roll back?
- How long does the rollback take?

"We can always roll back the deploy" is not a plan. Once you've dual-written for an hour, rolling back the *code* doesn't undo the *data*. The rollback for a data migration is itself a migration, and it needs the same expand-contract discipline.

Writing it first also catches the migrations that *can't* be rolled back cleanly — usually because step 5 has happened. Knowing that in advance changes how cautiously you ship step 4.

### Tooling

Pick one of:

- **Flyway** — SQL-first, opinionated, mature. The default for JVM shops.
- **Liquibase** — XML/YAML/JSON changesets, more abstract, supports more databases.
- **Sqitch** — Git-native, dependency graph between migrations, no ORM.

Which one matters less than the rules around it: version the migrations alongside the application code, run them as part of the deploy pipeline, never edit a migration after it has run anywhere. New change, new file.

Don't roll your own. The hard parts — ordering, idempotency, locking, recovery — are solved problems. The tool is the cheap part of the system.

### What This Costs

Expand-contract is more work than `ALTER TABLE`. A migration that would have been one PR becomes four. The calendar time stretches from a day to a week. The diff is bigger.

The pay-off is that no single step in the chain requires the database, the application, and the deploy to be perfect simultaneously. Each step is small, observed, and reversible. The migrations that page someone at 3am are the ones that tried to do everything at once.

### Bottom Line

Treat the migration as the unit of design, not the schema change. Expand-contract by default. Ship schema, code, and data separately. Flag the read path. Write the rollback first. Pick a tool and version it with the app. The database doesn't give you zero-downtime; the discipline does.
