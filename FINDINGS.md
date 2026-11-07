# Parmana security review: findings

Status: partial review. Source reading plus targeted follow-ups. The test suites were NOT run: execution was blocked by the auto-mode classifier. See "Update 2".

## Scope and method

- Repositories read: `parmana-core` (commit 19f74f2), `parmana-connectors` (cdb4975), `parmana-verifier` (707f4ad), `parmana-connect` (6966444). Verifier and connect were cloned but not reviewed beyond that.
- Method: read the production bootstrap, the execution gateway's verify and execute paths, and the connectors' release check, ledger, config and server. Read-only; no code was executed.
- Not done: running `npm test` in any repo. The install was blocked by the auto-mode classifier, and it was not worked around.
- Central question: can any path execute a real external action without verified business authority, authorization and approved executable content? No such path was found in the code read. The review is not complete.

## Verified controls

- **Production defaults are fail-closed (core).** Policy governance and signing readiness are relaxed only when `NODE_ENV` is `test` or `development` and an explicit opt-in is set. The nonce store is in-memory only under `NODE_ENV=test`. The late-bound signal verifier throws until bound.
- **Gateway verification order (core, `packages/execution-gateway/src/ExecutionGateway.ts`).** Envelope, then executable content hash, then policy hash and governance, then signal hash and divergence, then revocation, key scope, business conditions and authority grant. The nonce is consumed last. `execute()` dispatches only when `valid` is true. A missing `signalsHash` is refused when the signal verifier is wired.
- **Release signature (connectors, `src/service/release.ts`).** The Ed25519 signature covers the full canonical release. The SHA-512 commitment is accepted only above 4 KB. Audience, capability, `issuedAt`, `expiresAt` and lifetime (max 60 s) are checked.
- **Release ordering (connectors, `src/service/server.ts`).** The release is checked, then claimed in the ledger, then sent to the connector. A repeat of the same `businessTransactionId` returns the stored answer, or 409 while in progress.
- **Config (connectors, `src/service/config.ts`).** Requires an HTTPS audience and an Ed25519 public key. The nonce store defaults to Postgres.
- **Refund idempotency (connectors, Razorpay).** The refund receipt is the `businessTransactionId`, and an existing refund is checked before a new one is created.

## Findings

### F1. Incident runbook queries the wrong table (operational risk, high impact). Fixed in `parmana-connectors` branch `claude/sweet-clarke-3nsvst`

- `parmana-connectors/RUNBOOK.md` around line 334 tells operators to run `SELECT ... FROM parmana_connectors_nonces WHERE nonce = 'release:<businessTransactionId>'`. It says no row means the connector never ran, and that rows are purged.
- The live service uses `PostgresReleaseLedger`, which writes to `parmana_connectors_release_ledger` (`src/service/release-ledger.ts`). That table has no delete path.
- `src/service/nonce-store.ts`, which owns `parmana_connectors_nonces`, has no importers.
- **Failure scenario:** during a refund incident, an operator finds no row in the table the runbook names, concludes the connector never ran, and issues a second refund.
- **Recommendation:** correct the runbook to query `parmana_connectors_release_ledger` by `business_transaction_id`, and remove or label the dead nonce store.

### F2. The live Postgres release ledger has no tests (test coverage gap, high). Test added, not yet run

- `PostgresReleaseLedger` is the production default, but no test imports it. `tests/service/server.test.ts` uses `MemoryReleaseLedger`.
- `tests/service/nonce-store.postgres.test.ts` exercises the dead `PostgresNonceStore`, which gives false assurance about replay protection.
- **Failure scenario:** a regression in the `ON CONFLICT DO NOTHING` claim or the answer lookup would not be caught, and replay protection could break silently.
- **Recommendation:** add a Postgres integration test for the claim, the repeat answer, and the in-progress 409.

### F3. Replay identity is keyed on `businessTransactionId` only (resolved, was medium)

- The connector ledger keys on `businessTransactionId`, not `authorizationId`. A repeat with the same transaction id never executes twice.
- If core issued two releases with different transaction ids for one authorization, the connector would execute both. Core's `ExecutionControlService` release verifier should prevent that by consuming the nonce, but that was not verified.
- **Resolved.** Core's `createReleaseVerifier` (`packages/api/src/bootstrap/createReleaseVerifier.ts`) consumes the authorization's nonce once, on the single release path. The `businessTransactionHash` that the authorization signs covers the executable content, including `businessTransactionId`. So one authorization cannot yield two transaction ids, and the connector's keying on transaction id is safe. Keying on `authorizationId` as well would be defense in depth, but it is not required.

### F4. Core replay integration test is skipped without a database (test coverage, medium)

- `packages/api/tests/integration/replay.integration.test.ts` uses `describe.skipIf(!databaseConfigured)`. A green run without a database does not exercise replay.
- **Recommendation:** run this test in CI against a real Postgres, and report skipped counts.

### F5. Count correction to the preliminary report (documentation accuracy, low)

- The preliminary report said `parmana-core` has 866 TypeScript files. Counting `.ts` files excluding `node_modules` gives 1,283. The 866 figure may refer to a subset. Not reconciled.

## Open items

- Run the test suites in all four repos, and report pass, fail and skipped counts. Blocked; see "Update 2".
- Review the `ExecutionTrustApplication` recovery path.
- Review `parmana-verifier` (JS/Python parity, canonicalization, signatures) and `parmana-connect` (MCP, credentials, failure recovery).
- Review each connector adapter's capability set, idempotency and authentication.
- Cross-repository trace of one transaction from agent request to evidence.

## Update 1: fixes and follow-ups

- **F1 fixed** (`parmana-connectors`, `RUNBOOK.md`, branch `claude/sweet-clarke-3nsvst`). Step 2 now queries `parmana_connectors_release_ledger`. The runbook states that no row means the connector did not run in that database, and that a NULL answer needs step 3. The false "rows are purged" claim is removed. The 409 refusal message is quoted exactly.
- **F2 test added** (`parmana-connectors`, `tests/service/release-ledger.postgres.test.ts`). It covers the claim, the repeat answer across instances, and the concurrent claim (exactly one wins). It is skipped unless `PARMANA_TEST_DATABASE_URL` is set. **It has not been run.**
- **F3 resolved** (see above).

## Update 2: test run blocked

- `npm ci` in `parmana-connectors` succeeded.
- `npx vitest run` was denied by the auto-mode classifier. Pass, fail and skipped counts are therefore **unknown** for all four repos.
- Both the first run and the second run after your "go" were denied. I did not work around the denial.
- Item to unblock: allow `npx vitest run` (or `npm test`) for the connectors and core repos.
