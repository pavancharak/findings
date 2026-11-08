# Parmana security review: final report 2

Status: final for the matters below. This report covers what `FINAL-REPORT.md` (dated 7 November 2026) does not: the F26 and F28 series in the connectors, the condition enforcement work (F28-7), the refund payment-check gap found on 8 November, the signal design, the verifier integrity analysis, operational events, and test results since 7 November.
Date of this report: 8 November 2026.
Repositories: `pavancharak/parmana-core`, `pavancharak/parmana-connectors`, `pavancharak/parmana-connect`, `pavancharak/parmana-verifier`, `pavancharak/findings`.
Companion files: `FINAL-REPORT.md` (the first report), `FINDINGS.md` (Updates 1 to 42), `FINDINGS-2.md` (Updates 43 to 48).

## 1. Summary

- **Most important new finding (high).** No payment check runs in production before a refund. Customer-refund policy 1.2.0 takes `refundEligible` and `fraudCheckPassed` from the caller, and neither is checked against the payment. The connector's payment check (F28-7) runs only when a release carries the `payment.refundable` condition, and nothing in parmana-core or parmana-connectors creates that condition.
- **Business-condition enforcement (F28-7) is merged end to end** in both repositories, except the Trust Record test in the runtime package and the connector-side refund requirement, which is held (see section 4).
- **The connectors' Paytm refund and generic connector do not bind the release target (F28-4).** Still open, with no fix branch.
- **`parmana-connect` release path (F28-6)** still omits `target` and `businessTransactionId` and does not check `connectorId`, as last recorded. Not re-verified in this report.
- **Nothing in this report changes a production setting.** No policy has been published, no key has been created, no service has been deployed.

## 2. Findings not in the first report

| # | Finding | Severity | Status | Where |
|---|---|---|---|---|
| F26-5 | Paytm refund: whether a reused `refId` is rejected is unconfirmed. The connector retries a refund whose status is `UNKNOWN` with the same `refId`, which is safe only if Paytm rejects duplicates. | Medium until confirmed | Open. The Paytm docs saved on 8 November do not contain the duplicate-rejection code. Live Paytm refunds must stay off. | parmana-connectors, Paytm adapter |
| F26-8 | Paytm endpoint paths not confirmed against Paytm's documentation | Medium | Open, blocked on Paytm's documentation | parmana-connectors, Paytm adapter |
| F26-6 | Zoho create has no connector-level duplicate check | Low to medium | Open, design note | parmana-connectors, Zoho adapter |
| F26-7 | Zerodha paper orders are not guarded | Low | Open, design note. Not live | parmana-connectors, Zerodha adapter |
| F28-1 to F28-3 | Release path fixes: connector identity and conditions refusal | High, were blocking | Fixed and merged (parmana-connectors #49, #50) | parmana-connectors |
| F28-4 | Target ignored by several write adapters: the generic connector and Paytm refunds never read `target`; Zoho create and Zerodha do not check it. The signature covers both `target` and parameters, so this is a binding gap, not a forgery. | Medium | Open, no fix branch. HubSpot checks the target only when present. | parmana-connectors, Update 33 |
| F28-5 | Core signs `connectorId` as `ext-<capability>`; the connectors checked the vendor name. Every `razorpay:refund` release would have been refused, a regression from the F28-1 fix. | High | Fixed in PR #51 and in production (Update 35). The lesson: test a fix against the other side's real output. | parmana-connectors, Core |
| F28-6 | `parmana-connect` release path omits `target` and `businessTransactionId` and does not check `connectorId`. The Razorpay adapter refuses those releases. | Medium | Open as last recorded (Update 36). Not re-verified here. | parmana-connect |
| F28-7 | Business conditions were not enforced by the connectors service | High | Merged, see section 3 | parmana-connectors, parmana-core |

## 3. Business-condition enforcement (F28-7), current state

Merged in parmana-connectors (main at `eeac88a`):

- **Refusal of unverifiable conditions.** A release that carries a condition with no evaluator is refused before it is claimed, so it is not recorded and can be sent again (PR #52).
- **Evaluator framework.** Each signal key has one evaluator. Covered conditions are checked after the claim and before the connector runs (PR #55).
- **First evaluator and production wiring.** `payment.refundable` for `razorpay:refund`, registered in `src/service/condition-evaluator-registry.ts` (PRs #56, #57).
- **Evidence on both outcomes.** A condition that does not hold, or cannot be verified, still returns the check it observed, so the reason is recorded (PR #58).

Merged in parmana-core (`feat/external-conditions-evidence-record`, PR #503, merged 8 November 02:46 UTC, head `563161a`):

- Core validates `conditionsChecked` from the endpoint, refusing an invalid entry (outcome, atomicity, evidence shape, evidence size).
- Core keeps the validated checks in the answer.
- Credential-shaped evidence key names are refused, not recorded.
- A test shows a failed condition stops the refund: no connector call, and the failure is recorded in the answer and the evidence.
- Evidence is bound into the connector evidence hash, so changing it changes the hash the Trust Record anchors.

CI on `main` for the merge (run 42439915023, commit `3bbd4c9`) passed on every job, including the Test step with the v3 fixture test on Node 24.

Not done:
- **Trust Record end-to-end test (runtime package).** No test yet builds a full Trust Record from a Core-signed release, verifies it with `parmana-verifier`, and checks that tampering with the condition evidence fails verification. The gateway package does not depend on the runtime package, so the test belongs in runtime.
- **Connector refund requirement (PR #59).** A `razorpay:refund` release is refused unless it names `payment.refundable`. Held as a draft, and must not merge until Core produces that condition (see section 4).

## 4. Refund payment check: design, status, and sequencing

### The gap

- Customer-refund 1.2.0 (`policies/customer-refund/1.2.0/policy.json`) binds only `refundAmount` (from `parameters.amount`) and `managerApproved`. `refundEligible` and `fraudCheckPassed` are declared by the caller. The policy's own notes say nothing checks them against the order or a fraud system.
- Core copies business conditions into the signed authorization from validated signals (`RuntimeEngine.ts`). Nothing produces a `payment.refundable` signal.
- Result: a refund's eligibility is not checked against the payment. Manager approval is the only control on the amount.

### Design approved (8 November)

- A separate **Razorpay signal service**, in parmana-connectors, answers Core's signed queries. It holds the read-only Razorpay key, so Core never does.
- **Core's business sources are HTTP endpoints** registered through maker-checker, not in-process code.
- The policy declares `payment.refundable` from that source with `parameters: ["amount"]` and makes it required.
- The connector's release-time check (PR #59) stays as the second layer.

### Branches and status

| Branch | Repository | Commit | Status |
|---|---|---|---|
| `feat/razorpay-payment-signal-service` | parmana-connectors | `06da4d2` (unsigned) | Signal service: verifies Core's signed query (signature, audience, source, 60-second lifetime, nonce), reads the payment with a read-only key, signs its answer. A query with no amount gets `not_found`, which fails closed. Configuration required and checked at startup. 18 tests pass. Not deployed. |
| `feat/signal-query-parameters` | parmana-core | `0a2bf64` (unsigned) | A policy's `signalSources` entry lists the intent parameters a source needs. The signed query carries only those. Found in design: the query had no amount, so the signal service could not check one. 75 tests pass in the changed files; the full suite passes except `whole-record-v3-offline`, which needs Node 24. |
| `feat/razorpay-payment-signal` | parmana-core | `070ac50` (signed off locally) | **Superseded. Do not merge.** It passes the whole intent to every source and holds an in-process source that is not the production path. |
| `fix/razorpay-requires-refundable-condition` | parmana-connectors | `4cf8f20` (signed off) | PR #59, **draft, held**. Merging before Core emits the condition would refuse every refund. |

### Sequencing

1. Sign off and open the parmana-core PR for `feat/signal-query-parameters`.
2. Sign off `06da4d2` in parmana-connectors.
3. Choose secret names, and set the four signal-service settings in its secret store.
4. Confirm whether Razorpay offers a read-only or restricted key. If not, record that the key has full access.
5. Maker-checker registration of the signal service's endpoint and public key.
6. Customer-refund 1.3.0 declares `payment.refundable` and requires it. Policy approval needed before it goes live.
7. Only after 3 to 6: merge PR #59.

### Residual risks

- **Check-then-act window.** Razorpay has no conditional refund, so the payment can change between the read and the refund. Mitigation: reconcile refunds against Razorpay's refund records. Recorded as a known gap.
- **Eligibility of the order.** The check confirms Razorpay's state, not that the refund is right. Manager approval still covers that.
- **Signal query replay.** The signal service does not store nonces. A replay only re-reads the payment, so the harm is limited.

## 5. Verifier integrity: what the signature does and does not prove

Reviewed against `parmana-verifier/src/verify.mjs`:

- **Schema version 3 covers all top-level Trust Record fields except** `signature`, `signatures`, `schemaVersion`, `verifications` and `receipts`. That includes `executions`, where connector evidence sits. Tampering with a validated `conditionsChecked` therefore breaks verification. No new signature format is needed.
- **Schema version 2 is refused by default.** It signs seven fields only.
- **Limit: integrity is not semantics.** The verifier checks hashes, signatures, trusted keys and hybrid requirements. It does not know whether an evaluator was trustworthy, whether a payment was eligible, whether a check was atomic with the vendor action, or whether the evidence describes the vendor's real state. A valid signature proves the record was not altered under the verifier's trust assumptions. It does not prove the business claim.

## 6. Test coverage gaps

- **parmana-connectors `tests/service/server.test.ts`** has no condition assertions. Condition coverage lives in `condition-evaluation.test.ts` and `condition-evaluator-registry.test.ts`.
- **parmana-core `external-adapter.test.ts`**: the test "records only the known fields of an answer" passes today because its answer has no `conditionsChecked`. Its name and contract are out of date now that condition evidence is part of the answer. It needs a case that includes condition evidence.
- **Trust Record end-to-end test** (section 3): not written.
- **Razorpay evaluator**: unit tests with a mocked `fetch` only. No test against Razorpay test mode.
- **Connectors test count:** 250 pass with 8 skipped, which need a database.

## 7. Operational events

- **Database credential exposed in chat (earlier in this review, recorded in FINDINGS Update 31).** A production connection string was pasted into a conversation. The owner rotated the password. The value is not recorded in any file here and must not be.
- **Deploy error.** The connectors' Vercel build failed with TS2688 because `@types/node` was a devDependency. Fixed by moving it to dependencies (PR #50).
- **Route error.** Vercel returned NOT_FOUND on `/execute`; the route is `/api/execute`.
- **Flaky test.** `release.test.ts` failed in CI on PR #49 because it read the clock twice. Fixed to read it once.
- **Scheduled task and environment mistakes** on the operator's machine (a placeholder value in an environment variable, and a racing task delete) were caught before they took effect and corrected.
- **Accidental commits of local files** (`.vercel/`, an env example) were removed by follow-up commits. No force push was used.

## 8. Tests run since the first report

| Repository | Node | Result |
|---|---|---|
| parmana-connectors (the connectors suite, after the held PR #59) | 22.23.4 | 234 pass, 8 skipped, no database |
| parmana-connectors (signal service branch) | 22.23.4 | 250 pass, 8 skipped, no database |
| parmana-core, after PR #503 (CI, Node 24) | 24.x | All jobs pass on `main` for `3bbd4c9`, including Test and replay integration against a migrated database |
| parmana-core, `feat/razorpay-payment-signal` (step 1) | 22.23.4 | Source tests 11 of 11; runtime unit tests 206 pass, 9 skipped |
| parmana-core, `feat/signal-query-parameters` | 22.23.4 | Full suite 3,522 pass, 148 skipped, 1 failed: `whole-record-v3-offline`, which needs Node 24 |
| parmana-core, `feat/evidence-key-names` and step 5 | 22.23.4 | Gateway suite 220 pass, 9 skipped |

Environment notes: the sandbox runs Node 22, while the project requires Node 24.x. The one failure above is from that. The sandbox rebuilt the `re2` native module, which is an environment fix only.

## 9. Pull requests and branches, status at 8 November

| PR | Repository | Status |
|---|---|---|
| #503 (feat/external-conditions-evidence-record) | parmana-core | Merged, 8 Nov 02:46 UTC |
| #17 (Update 19 to 22) | findings | Merged, 8 Nov 02:49 UTC |
| #18 (Update 47) | findings | Merged, 8 Nov 02:50 UTC |
| #19 (Update 48) | findings | Open |
| #59 (refund requires `payment.refundable`) | parmana-connectors | Draft, held. Do not merge |
| #58 (condition evidence on every outcome) | parmana-connectors | Merged (on `main` at `eeac88a`) |
| #502, #501 | parmana-core | Merged 7 Nov |

Branches with no PR yet: `feat/signal-query-parameters` (parmana-core), `feat/razorpay-payment-signal-service` (parmana-connectors). `feat/razorpay-payment-signal` (parmana-core) is superseded.

## 10. Corrections to earlier statements

- **F28-7 "step 2 in review" (FINDINGS Update 37)** was corrected in Update 38: merged.
- **"Production wiring missing"** was true before PR #57 and is fixed (Update 41).
- **"Core checks freshness only at dispatch"** is accurate for the release path: Core checks `validUntil` before signing.
- **"Connector identity mismatch"** was real and was fixed (F28-5).
- **Earlier in the 8 November session**, I described the Core change as "Core holds the Razorpay key", then replaced that design with the separate signal service. That design is the one approved and built.
- **Earlier in the 8 November session**, I wrote that the Core step 1 source was the production path. It is not; it is superseded by the signal service.

## 11. Open items

1. Sign off and open PRs for the two unsigned branches listed in section 9.
2. Maker-checker registration, policy 1.3.0 approval, and the read-only Razorpay key decision (section 4).
3. Merge PR #59 only after step 7 in section 4.
4. Trust Record end-to-end test in the runtime package (section 3).
5. Update `external-adapter.test.ts` for condition evidence (section 6).
6. F28-4: fix target binding for the generic connector and Paytm refunds.
7. F28-6: `parmana-connect` release path, `target`, `businessTransactionId` and `connectorId`. Re-verify first.
8. F26-5 and F26-8: Paytm duplicate-refId behaviour and endpoint paths, from Paytm's documentation. Live Paytm refunds stay off until then.
9. F26-6 and F26-7: design notes for Zoho create and Zerodha paper orders.
10. Trace the gateway attestation and direct `ExecutionControlService.execute()` calls (F9 from the first report).
11. Decide retry or reconcile for a `begin()` failure (F8), and confirm session consumption by issuing instance (F10).
12. Separate repository errors from "policy not found" in gateway diagnostics (F11).
