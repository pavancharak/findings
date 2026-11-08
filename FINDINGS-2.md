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
