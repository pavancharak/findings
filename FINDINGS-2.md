# Parmana security review: second findings file

Updates 43 to 46 are kept here, not in FINDINGS.md, so that FINDINGS.md matches main. Updates 19 to 42 are in FINDINGS.md on main.

## Update 43: F28-7 condition enforcement, parmana-core follow-ups

Scope: the two open items in parmana-core on top of PR #503 (`feat/external-conditions-evidence-record`, commit 2177f4d). Both were implemented and tested in a sandbox worktree. Neither is merged.

### Evidence key names (F28-7, key-name follow-up)

- Problem: the evidence check validated values but not key names. A credential-shaped key inside `conditionsChecked[].evidence`, such as `apiToken`, was accepted and would be recorded in the Trust Record.
- Change: `GatewayExternalAdapter.ts` refuses evidence with a key matching `credential|secret|token|apikey|api_key|password|authorization`, whatever its value. The refusal message is `…evidence has a credential-shaped key "<key>"`.
- Test: `external-adapter-conditions.test.ts`, "refuses evidence with a credential-shaped key, however its value looks".
- Patch: `core-keys-evidence-key-names.patch`. It applies on top of 2177f4d and touches two files.

### End-to-end path (F28-7, step 5)

- Change: new test file `packages/execution-gateway/tests/unit/condition-refund-path.test.ts`.
- What it covers, in one flow:
  1. The release Core signs verifies with `verifyParmanaRelease` and carries the business condition.
  2. The documented endpoint finds the condition no longer holds, so it does not act. The adapter reports `success: false` with `conditionsNotMet`.
  3. `buildConnectorEvidence` built from the adapter's real response records the `not-holding` outcome and its evidence.
  4. A held condition and a failed one produce different evidence hashes.
- Not covered: the full Trust Record. It is assembled in the runtime package, which the gateway package does not depend on. A test for that assembly would go in runtime.
- Note: the endpoint's `conditionsChecked` reply is shaped inside the test. The documented example endpoint does not send one yet.
- Patch: `core-keys-step5.patch`. It adds one file and applies after the key-names patch.

### Checks (sandbox, both patches applied)

- Execution-gateway suite: 220 passed, 9 skipped.
- Typecheck, lint, format, python-models: passed.
- Build and examples: passed.
- Full preflight test step: 3514 passed, 148 skipped, 1 failed.
  - The failure is `typescript/test/whole-record-v3-offline.test.ts`. The sandbox runs Node 22.23.4 and the project requires Node 24.x. Node 22 cannot parse the ML-DSA-65 key in the fixture. The test imports only the TypeScript SDK's offline verifier, not the gateway code, so these changes cannot cause it. Not yet confirmed on Node 24.
- The sandbox rebuilt the `re2` native module, which the earlier install had skipped. This was an environment fix, not a repository change.

### Status

- Local copy in parmana-core: the key-names patch was applied on the user's machine on `feat/external-conditions-evidence-record`. Commit and push status: not yet confirmed.
- Step 5 test: not yet applied or committed on the user's machine.
- Not merged: parmana-core PR #503 is still open.

### Still open

- Commit and push the key-names change and the step 5 test to PR #503.
- Confirm the v3 fixture test passes on Node 24 on the user's machine.
- Full Trust Record test in the runtime package.
- The example endpoint does not yet send `conditionsChecked`.

## Update 44: F28-7 steps 3 to 5 merged

- parmana-core PR #503 (`feat/external-conditions-evidence-record`) merged 2026-11-08 02:46 UTC. Head `563161a`.
- Commits on the merged branch, each with the author's sign-off: `2177f4d` (steps 3 and 4), `d24ff7a` (credential-shaped evidence key names), `563161a` (end-to-end test that a failed condition stops the refund).
- Update 43 described these as unpushed and unmerged. That status is now superseded by this update.
- Not yet verified: the Node 24 CI result for the v3 fixture test on the merged branch.

## Update 45: CI results for F28-7 steps 3 to 5

- CI on the PR head `563161a` (parmana-core run 42439658962): **success**. This covers the step 5 test and the Node 24 run of the v3 fixture test that failed on the sandbox's Node 22.
- CI on the three PR commits: `2177f4d` (run 42438179324), `d24ff7a` (run 42439163084): both **success**.
- CI on the merge commit on `main` (`3bbd4c9`, run 42439915023): **in progress** when checked. Not yet confirmed.
- Resolved from Update 44: the v3 fixture test's Node 24 result is now known (passes in CI).

### Still open

- Confirm the `main` CI run for `3bbd4c9` finishes green.
- Full Trust Record test in the runtime package (not started).
- The example endpoint does not yet send `conditionsChecked`.

## Update 46: CI on main, in progress

- CI run 42439915023 (merge commit `3bbd4c9` on `main`), job `build-and-test`: passed Install, Build, Configuration doctor, Lint, Typecheck, Format and Guard against retired terminology. Test step **in progress** at 02:47 UTC.
- Job `verify-policy-approvals` and job `python-reference-drift`: both **success**.
- Still pending: the Test step (which includes the v3 fixture test on Node 24), replay integration, and policy approval log steps.
- Findings PR: #17 (`claude/sweet-clarke-3nsvst` into `main`).

## Update 47: CI on main for the F28-7 merge, final result

- CI run 42439915023 (merge commit `3bbd4c9` on `main`, parmana-core): **success**. All jobs passed: build-and-test (Build, Lint, Typecheck, Format, Guard against retired terminology, Test, Replay integration against a migrated database, Verify the policy approval log), verify-policy-approvals and python-reference-drift.
- The Test step, which includes the v3 fixture test on Node 24, passed on `main`. This closes the open item from Updates 44 to 46.
- Update 46 said the Test step was still running. That is now superseded by this update.
- Remaining open items from Update 43: the full Trust Record test in the runtime package, and the example endpoint's `conditionsChecked` reply.

## Update 48: refund payment check, design and status

### Merged

- parmana-core PR #503 (merged 2026-11-08 02:46 UTC, head `563161a`): steps 3 to 5 of F28-7. Evidence key names are checked, and an end-to-end test shows a failed condition stops the refund. CI on `main` passed, including the Node 24 run of the v3 fixture test.

### Finding: no payment check runs in production today

- `policies/customer-refund/1.2.0` binds only `refundAmount` and `managerApproved`. `refundEligible` and `fraudCheckPassed` are declared by the caller, and the policy itself says nothing checks them.
- Nothing in parmana-core or parmana-connectors creates a `payment.refundable` condition. The connector's evaluator therefore runs only when an authorization happens to carry that condition.
- Severity: high. A refund's eligibility is not checked against the payment. The manager approval is the only control on the amount.

### Connector gap, held

- parmana-connectors PR #59 (`fix/razorpay-requires-refundable-condition`): a `razorpay:refund` release is refused unless it names its `payment.refundable` condition. Commit `4cf8f20`, signed off, DCO passed.
- **Held as a draft, title marked do not merge.** Core does not send that condition yet, so merging now would refuse every refund.

### Design, approved

The check runs in a separate Razorpay signal service, and parmana-core never holds the payment key. Core's business sources are HTTP endpoints that answer signed queries, registered through maker-checker.

Branches, with status:

- **parmana-connectors `feat/razorpay-payment-signal-service`** (`06da4d2`, unsigned). The signal service: verifies Core's signed query (signature, audience, source, 60-second lifetime, nonce), reads the payment with a read-only key, and signs its answer. A query with no refund amount is answered `not_found`, which fails closed. Configuration is required and checked at startup. 18 tests pass. Not deployed.
- **parmana-core `feat/signal-query-parameters`** (`0a2bf64`, unsigned). A policy's `signalSources` entry can list the intent parameters a source needs. The signed query carries only those, never the whole intent. Found while designing: the signed query had no amount, so the signal service could not check one. 75 tests pass in the changed files. The full suite passes except `whole-record-v3-offline`, which fails only on the sandbox's Node 22 and needs Node 24.
- **parmana-core `feat/razorpay-payment-signal`** (`070ac50`, signed off locally by the user). **Superseded. Do not merge.** It passes the whole intent to every source, and it holds an in-process Razorpay source that is not the production path.

### Open decisions and actions

1. Sign off `0a2bf64` in parmana-core, then open its PR.
2. Sign off `06da4d2` in parmana-connectors (the DCO check needs it).
3. Choose the secret names and set the four signal-service settings in its secret store.
4. Read-only Razorpay key: confirm whether Razorpay offers a read-only or restricted key. If not, record that Core's key is full access.
5. Maker-checker registration of the signal service's endpoint and public key.
6. Customer-refund 1.3.0: declare `payment.refundable` from the signal service with `parameters: ["amount"]`, and make it required. Approval is needed before it goes live.
7. Only after 3 to 6: merge connectors PR #59.
8. Sequencing: Core change, signal service deployed, policy 1.3.0 approved and live, then PR #59.

### Test and environment notes

- Sandbox runs Node 22; the project requires Node 24.x. The `whole-record-v3-offline` failure is environmental.
- The sandbox rebuilt `re2` for Node 22. This is an environment fix, not a repository change.

### Residual risks

- **Check-then-act window.** Razorpay has no conditional refund, so the payment can change between the read and the refund. Mitigation: reconcile refunds against Razorpay's refund records. Keep as a known gap.
- **Eligibility is a fact about the payment, not the order.** The check confirms Razorpay's state, not that the refund is right. Manager approval still covers that.
- **Replay of signal queries.** The signal service doesn't store nonces. A replayed query only re-reads the payment, so the harm is limited, but it should be noted.

### Still open from earlier updates

- parmana-core `external-adapter.test.ts`: the test named "records only the known fields of an answer" passes today because its answer has no `conditionsChecked`. Its name and contract no longer match once condition evidence is part of the answer. Needs a case that includes condition evidence.
- Full Trust Record test in the runtime package.
- The example endpoint does not send `conditionsChecked`.
