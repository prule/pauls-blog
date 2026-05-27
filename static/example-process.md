```text
/opsx:propose
```

What change do you want to work on? Describe what you want to build or fix.

```text
fund-transfer (F06) (Recommended)
```

All four artifacts created for change `fund-transfer` at [openspec/changes/fund-transfer/](openspec/changes/fund-transfer/):

- **[proposal.md](openspec/changes/fund-transfer/proposal.md)** — F06 composes F01 (Account mutators + exceptions), F02 (ledger/journal port), F05 (Accounts port + 404 mapping), and F07 (paired-lock primitive) into the service's first HTTP write endpoint. Closes the `self-transfer` open decision by rejecting same-account transfers as `400 BAD_REQUEST_PAYLOAD` via a new `SameAccountTransferException`. Adds four `@ExceptionHandler` entries for the F01 domain exceptions.
- **[design.md](openspec/changes/fund-transfer/design.md)** — Key decisions: `@Transactional` on the controller method (only controller in the codebase with it); **lock-then-load** to prevent stale aggregates; `Money.of(...)` rounds silently to 2 decimals; bean validation handles missing-field + non-positive amount via F03's existing handler; injectable `Clock` for test determinism; one handler entry per exception type (different codes, different messages).
- **[specs/fund-transfer/spec.md](openspec/changes/fund-transfer/specs/fund-transfer/spec.md)** — 7 `ADDED` requirements: POST endpoint shape (with `operationId: createTransfer`), payload validation (incl. self-transfer rejection), business-rule error mapping (404/ACCOUNT_INACTIVE/INSUFFICIENT_FUNDS), atomicity of all four writes, journal description/status/timestamp, lock-then-load ordering, transactional boundary on the controller, and `SameAccountTransferException` placement.
- **[tasks.md](openspec/changes/fund-transfer/tasks.md)** — 9 sections (~32 tracked tasks): domain → application use case → OpenAPI → controller → handler → wiring → tests → manifest → verification.

All artifacts complete! Run `/opsx:apply` to start implementing.

```text
/opsx:apply
```

Using change: `fund-transfer`. Override with `/opsx:apply <other>`.

## Implementing: fund-transfer (schema: spec-driven)

I have propose-phase context. Reading the few files I'll need to extend.

Got the patterns. Starting implementation.

## Task 1.1–1.3: domain

## Task 2: application use case

Now the unit tests (2.3, 2.4).

## Task 3: OpenAPI contract additions

`amount` generated as `BigDecimal` with `@DecimalMin("0.01")`, account fields with `@NotNull @Size(min = 1)`. Task 3 done.

## Task 4: controller + mapper

## Task 5: GlobalExceptionHandler extensions

## Task 6: bean wiring

## Task 7: integration tests

Spring Boot 3.4.5 — `@MockitoSpyBean` is supported. Cleaning up unused imports.

Now the controller integration test (7.2).

Task 7.3: extend `OpenApiContractTest`.

## Task 8: close `self-transfer` open decision

## Task 9: verification

Application module has no assertj — using junit + mockito only.

ByteBuddy doesn't support Java 25 yet. Need to set `net.bytebuddy.experimental=true`. Checking the parent build for a place to set this globally.

Now bootstrap.

Bootstrap main scope needs explicit `application` and `domain` deps since `infrastructure` exposes them via `implementation` (not `api`).

All bootstrap tests green. Now clean build + bootRun curl checks.

All checks pass:
- `operationId: createTransfer` ✓
- Empty body → 400 BAD_REQUEST_PAYLOAD with field names ✓
- Nonexistent accounts → 404 RESOURCE_NOT_FOUND (exercises lock-then-load + 404 path end-to-end) ✓

Stop the app + mark tasks complete.

## Implementation Complete

**Change:** fund-transfer
**Schema:** spec-driven
**Progress:** 34/34 tasks complete ✓

### Files created
- [domain/.../SameAccountTransferException.java](domain/src/main/java/com/bank/core/domain/SameAccountTransferException.java) + [test](domain/src/test/java/com/bank/core/domain/SameAccountTransferExceptionTest.java)
- [application/.../transfer/](application/src/main/java/com/bank/core/application/transfer/) — `TransferCommand`, `TransferFunds`
- [application/.../transfer/test/](application/src/test/java/com/bank/core/application/transfer/) — `TransferCommandTest`, `TransferFundsTest`
- [infrastructure/.../web/transfer/](infrastructure/src/main/java/com/bank/core/infrastructure/web/transfer/) — `TransferController`, `TransferRequestMapper`
- [openapi/paths/transfers.yaml](bootstrap/src/main/resources/openapi/paths/transfers.yaml) + [openapi/schemas/transfer-request.yaml](bootstrap/src/main/resources/openapi/schemas/transfer-request.yaml)
- [TransferAtomicityIntegrationTest](bootstrap/src/test/java/com/bank/core/persistence/transfer/TransferAtomicityIntegrationTest.java) + [TransferControllerTest](bootstrap/src/test/java/com/bank/core/web/transfer/TransferControllerTest.java)

### Files edited
- [openapi.yaml](bootstrap/src/main/resources/openapi/openapi.yaml) — registered `/api/v1/transfers` path + `TransferRequest` schema + `transfers` tag
- [GlobalExceptionHandler.java](infrastructure/src/main/java/com/bank/core/infrastructure/web/error/GlobalExceptionHandler.java) — 4 new `@ExceptionHandler` entries + Javadoc update
- [BankCoreApplication.java](bootstrap/src/main/java/com/bank/core/BankCoreApplication.java) — `@Bean Clock systemClock()` + `@Bean TransferFunds`
- [bootstrap/build.gradle.kts](bootstrap/build.gradle.kts) — added `implementation(project(":application"))` + `implementation(project(":domain"))` so `BankCoreApplication` can reference them in `@Bean` factories
- [build.gradle.kts](build.gradle.kts) — `systemProperty("net.bytebuddy.experimental", "true")` for Mockito on Java 25
- [OpenApiContractTest.java](bootstrap/src/test/java/com/bank/core/web/OpenApiContractTest.java) — 2 new assertions for `createTransfer` + `TransferRequest`
- [openspec/config.yaml](openspec/config.yaml) — `self-transfer` open decision closed

### Verification
- `:domain:test`, `:application:test`, `:bootstrap:test` all green
- `./gradlew clean build` green (25 tasks)
- bootRun live checks: `operationId=createTransfer`; empty body → 400 BAD_REQUEST_PAYLOAD with field-naming message; nonexistent accounts → 404 RESOURCE_NOT_FOUND end-to-end via lock-then-load

All tasks complete! You can archive this change with `/opsx:archive`.


```text
/opsx:archive
```

All done. Delta spec exists at `openspec/changes/fund-transfer/specs/fund-transfer/spec.md`. The main spec at `openspec/specs/fund-transfer/spec.md` has the original 5 requirements; the delta refines them with more detail and adds 3 new ones (lock-then-load ordering, controller-level transactional boundary, `SameAccountTransferException` placement).

Sync the delta spec into openspec/specs/fund-transfer/spec.md before archiving?

```text
Sync now
```

## Archive Complete

**Change:** fund-transfer
**Schema:** spec-driven
**Archived to:** [openspec/changes/archive/2026-05-24-fund-transfer/](openspec/changes/archive/2026-05-24-fund-transfer/)
**Specs:** ✓ Synced to [openspec/specs/fund-transfer/spec.md](openspec/specs/fund-transfer/spec.md) — 8 requirements (5 refined from the prior baseline + 3 new: lock-then-load ordering inside the use case, controller-level transactional boundary, `SameAccountTransferException` placement). Purpose paragraph extended to call out the composition over F02/F05/F07.

All artifacts complete. All 34 tasks complete.

```text
commit
```

Four uncommitted phases stacked together. I'll split into 2 commits — first the account-lookup archive, then everything F06 (propose+apply+archive squashed since no intermediate commits were made).

Now everything F06 in one commit (propose+apply+archive squashed — no intermediate commits were made between phases).

Two commits:
- `0e2a218 opsx:archive account-lookup` (renames + spec sync)
- `22497f2 opsx:apply fund-transfer (propose+apply+archive squashed)` (25 files, +1712/−41) — code, build, openspec docs in archive, and spec sync. Squashed because no intermediate commits were taken between F06's propose, apply, and archive phases.

Working tree clean.
