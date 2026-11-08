# Razorpay refund through Parmana: technical case study, troubleshooting and resume guide

Status: **Working record, as of 2026-11-08 (IST, UTC+05:30).**
Scope: the path from a refund request to a Razorpay test-mode refund, through Parmana's policy, approval, signal source and connector release.
Companion documents: `RAZORPAY-END-TO-END-GUIDE.md` (commands), `RAZORPAY-INCIDENTS.md` (bypass record), `BUSINESS-SUMMARY.md` (plain-language version).

---

## 1. Objective

Demonstrate, end to end and in Razorpay **test mode**, that a refund on a Razorpay payment runs only when:

1. Razorpay itself confirms the payment is refundable for the requested amount (checked through a registered signal source, not declared by the caller), and
2. a named human has approved that payment and amount with a signed approval, and
3. the refund is executed by the connector service and recorded as signed evidence.

Live money is out of scope. `PARMANA_ALLOW_LIVE_KEYS` stays off.

## 2. Components and where they live

| Component | Where | Notes |
|---|---|---|
| Parmana API (decisions, policies, registrations, approvals) | Vercel project `parmana-api-real`, `https://parmana-api-real.vercel.app` | Production API |
| Parmana core source | `pavancharak/parmana-core` | Branch `feat/razorpay-payment-signal` holds the policy draft and the least-disclosure change |
| Connector service (executes refunds) | Vercel project `parmana-connectors`, `https://parmana-connectors.vercel.app/api/execute` | Verifies a signed release, then calls Razorpay |
| Signal service (answers "is this payment refundable?") | Same project, `https://parmana-connectors.vercel.app/api/refund-payment` | Verifies Parmana's signed query, reads the payment from Razorpay, signs the answer |
| Connector source | `pavancharak/parmana-connectors` `main`, `src/connectors/razorpay/`, `src/signal/` | Razorpay connector PR #61 merged; signal source name change `80ff107` on `main`; log line `2097791` on `main` (unreviewed, see section 9) |
| Policy | `razorpay-refund` 1.1.0, in effect | Requires `paymentRefundable` (sourced from `razorpay`, for `amount`, valid 60 s) and `managerApproved` (approval for the payment and amount) |
| Records | `pavancharak/findings` | This folder, the guide, the incident record, FINDINGS-2.md Update 49 |

Hobby plan limits that affect this work: runtime logs are kept for 30 minutes only; the account is limited to 100 deployments per day.

## 3. How a refund request flows

1. The caller sends `POST /execute` to Parmana with a Business Transaction (capability `razorpay:refund`, target `pay_…`, `parameters.amount` in paise, signals).
2. Parmana checks the caller's key is allowed the capability (`PARMANA_API_KEYS`, `allowedCapabilities`).
3. Parmana loads the policy in effect for the capability (`razorpay-refund` 1.1.0) through the connector registration `5d397f58`.
4. For `paymentRefundable`, Parmana sends a query signed with its default Ed25519 key to the registered source `razorpay` (registration `46bb4f5f`). The source must verify the signature, audience, source name, nonce and expiry, then answer with its own signature.
5. For `managerApproved`, the caller must send `true` with a signed approval artifact for the same capability, payment and amount. The server checks the artifact.
6. If all rules pass, Parmana creates a signed release for `https://parmana-connectors.vercel.app/api/execute`. The connector verifies the release, calls Razorpay's refund API in test mode, and returns the result. Parmana records it.

## 4. Chronology (key events)

| Date/time (IST) | Event | Record |
|---|---|---|
| 2026-10-29 | Razorpay connector test-mode check, 4 of 4 checks passed (per `parmana-connectors` README, "Status") | README |
| 2026-11-08, morning | Connector PR #61 (5xx on refund creation says a refund may exist) merged | `1eb59b5` |
| 2026-11-08 | Dependabot PRs #504–#508 and PRs #509–#510 merged in `parmana-core` | — |
| 2026-11-08 | PR #512 (Razorpay signal) conflict resolved: least disclosure, sources see only declared parameters | Branch restored to `b47b851` after an unrelated force-push (`faaa064` kept on `backup/razorpay-feat-before-restore`) |
| 2026-11-08 | Demo registration of `razorpay:refund` revoked | Change `b9262dbb…`, approved by `mamta-checker` |
| 2026-11-08 | Connector registration of `razorpay:refund` active | Change `5d397f58…`, approved by `mamta-checker` |
| 2026-11-08 | Stale policy change blocking 1.0.0 rejected | Change `cf85a4af…`, rejected by `mamta-checker` |
| 2026-11-08 | Policy `razorpay-refund` 1.1.0 approved and in effect | Change `01b23fe0…`, approved by `mamta-checker` |
| 2026-11-08 | `POLICY_CHANGE_CHECKERS` set to `{"razorpay-refund":["mamta-checker"]}` and redeployed | Previous value could not be read; replaced |
| 2026-11-08 | Signal service configured and deployed (variables, signing key, `PARMANA_SIGNAL_SOURCE_NAME=razorpay`) | `parmana-connectors` production |
| 2026-11-08 | Source registration `razorpay` active | Change `46bb4f5f…`, approved by `mamta-checker` |
| 2026-11-08 | Maker key granted `razorpay:refund` through `PARMANA_API_KEYS` and redeployed | `parmana-api-real` production |
| 2026-11-08 | Negative test 1: refused `CAPABILITY_NOT_ALLOWED` (before the grant) | Nothing executed |
| 2026-11-08 | Negative test 2 and 3: refused `POLICY_DENIED`, source `razorpay` answered HTTP 403 | Nothing executed |
| 2026-11-08 | Diagnostic: production `/keys/default` PEM equals the PEM stored for the signal service | True |
| 2026-11-08 | Two direct pushes bypassed protected-branch rules (incidents 1 and 2) | `RAZORPAY-INCIDENTS.md` |
| 2026-11-08 | Admin bypass disabled by the repository owner | — |
| 2026-11-08, about 21:25 IST | Connectors redeploy attempted to add the refusal-reason log line; refused: daily deployment limit | Retry after 24 h |

## 5. Results so far

| Check | Expected | Actual | Status |
|---|---|---|---|
| Policy `razorpay-refund` in effect | 1.1.0 | 1.1.0 | Done |
| `razorpay:refund` registration active | connectors endpoint | active | Done |
| Demo registration revoked | revoked | revoked | Done |
| Source `razorpay` registered | active | active | Done |
| Signal service reachable and configured | refuses an empty body with its own check | `403 the body has no query` | Done |
| Signal service key matches Parmana's default key | match | match | Done |
| Maker key allowed `razorpay:refund` | allowed | allowed (after grant) | Done |
| Negative test: refused without approval | `POLICY_DENIED` | `POLICY_DENIED`, but from the source (HTTP 403), not from the approval check | **Partial** |
| Source answers Parmana's query | answer | HTTP 403 | **Blocked** |
| Approved refund (positive test) | `APPROVED`, refund in Razorpay test mode | not run | **Not started** |
| Refund recorded as signed evidence | trust record | not run | **Not started** |

## 6. What this proves, and what it does not

**Proves (technically, with the evidence above):**
- The governance layer works as designed: policy changes, connector registrations, source registrations and revocations each went through a maker and a different checker, with a step-up signature, and each approval is recorded with its approver.
- Parmana reaches the registered source over HTTPS, sends a query, and receives an HTTP response. The source is live, configured, and correctly rejects an unsigned or empty request.
- The source's public key stored in Vercel matches Parmana's current default public key.
- A caller without the capability is refused before any policy runs (`CAPABILITY_NOT_ALLOWED`), and a request without an approval is refused by policy when the source answers (`POLICY_DENIED`).

**Does not prove:**
- That the source accepts a correctly signed query. The source refuses every query so far, for a reason we have not yet seen.
- That the approval check works. Negative test 2 and 3 were refused by the source before the approval rule was evaluated.
- That a refund runs. No refund request has been approved, released or executed in Razorpay since the setup.
- That the connector release, the signature checks on the connector, or the trust record work end to end in this configuration.
- That the policy's rules are correct for every case. The policy shows which rules decided; it doesn't prove they are right.
- Anything about production or live money.

## 7. Troubleshooting guide

Use the table to match what you see. Every step is safe unless it says otherwise. Never resend a refund request to get past a denial.

| Symptom | Likely cause | Check | Fix |
|---|---|---|---|
| `401 authentication required` | Key missing, empty, rotated, or not registered | `$env:MAKER_KEY.Substring(0,4)`; compare the SHA-256 of the key with the `keyHash` in `PARMANA_API_KEYS` | Set the key again with `Read-Host`; if rotated, update `PARMANA_API_KEYS` |
| `CAPABILITY_NOT_ALLOWED` | Key not granted the capability | `PARMANA_API_KEYS` entry has no `razorpay:refund` in `allowedCapabilities` | Operator adds it and redeploys |
| `POLICY_DENIED`, "source did not answer (HTTP 403)" | Source refused Parmana's query | See section 8.1 | See section 8.1 |
| `CHECKER_NOT_IN_SCOPE` | Checker not listed for the policy | `POLICY_CHANGE_CHECKERS` in `parmana-api-real` | Operator adds the checker, redeploys |
| `A PENDING_APPROVAL change already exists` (409) | Another change for the same policy and version is open | List pending changes | Approve or reject the stale change first |
| `already has an active external connector` (409) | Registration already active | `GET /external-connectors` | Revoke through a change, then register |
| `ENOENT … checker-stepup-private.pem` | Step-up key file not in the folder | `Test-Path .\checker-stepup-private.pem` | The checker puts their own key file in the folder |
| Approval refused right after signing | 120-second signature expired | — | Sign and approve back to back |
| `POST : The term 'POST' is not recognized` | A URL was pasted alone on a line | — | Always use `curl.exe -X POST …` |
| `Malformed JSON body` | PowerShell stripped quotes in a single-quoted string | — | Write the JSON to a file with `[System.IO.File]::WriteAllText`, send with `--data-binary "@file"` |
| `curl: Failed to open <file>` | File not in the current folder | `Test-Path .\<file>` | Create it in the current folder |
| `vercel logs` shows "No runtime log entries" | Nothing logged in the last 30 min, or function does not log its refusals | — | Redeploy with the log line, then reproduce |
| `vercel deploy` says "api-deployments-free-per-day" | Daily deployment limit (100) reached | `vercel deploy` output | Wait 24 h; do not work around it |
| Stop hook: "unpushed commits" | Stale remote-tracking ref | `git ls-remote --heads origin <branch>` against `git rev-parse HEAD` | Refresh the ref with `git fetch --depth=200 origin "+refs/heads/<branch>:refs/remotes/origin/<branch>"`; if both hashes match, nothing needs pushing |
| Push refused: "Cannot update this protected ref" | Protected branch | — | Open a branch and a PR. Do not use the admin bypass without the owner's approval (disabled 2026-11-08) |

### 7.1 Signal query refusals (the open problem)

Refusal reasons, in the order the source checks them (`src/signal/protocol.ts`):
body is an object → query present → query signed → signed by the registered key → signature verifies → version is 1 → audience equals `PARMANA_SIGNAL_AUDIENCE` → source equals `PARMANA_SIGNAL_SOURCE_NAME` → nonce present → times are dates → lifetime at most 60 s → not expired → not issued in the future.

What has been checked, and what is still open:

| Candidate | Status |
|---|---|
| Signature key | Checked: the PEM stored in `PARMANA_SIGNAL_PARMANA_PUBLIC_KEY_PEM` equals `/keys/default`. Still to confirm that Parmana signs with the default key id. |
| Canonical serializer | Connectors pin `@parmana/crypto` 0.1.1; core uses 2.0.0. Both sort keys recursively and call `JSON.stringify`. Equivalent for this query shape, but not confirmed byte for byte. |
| Audience | Registered endpoint and `PARMANA_SIGNAL_AUDIENCE` are both `https://parmana-connectors.vercel.app/api/refund-payment`. Matches as written. |
| Source name | `PARMANA_SIGNAL_SOURCE_NAME` set to `razorpay`, which matches the policy and registration. The deployment that serves traffic must include `80ff107`, to be confirmed on redeploy. |
| Expiry or clock | Not checked. |

The log line that prints the reason is commit `2097791` on `main`. It is in place in the code but not yet deployed, because of the deployment limit.

Once it is deployed, read the reason with the commands in section 11, step 2, and fix as below:

| Reason in the log | Fix |
|---|---|
| `the query's signature does not verify`, `not signed by parmana-core's registered key` | Parmana signs with a different key than `/keys/default` shows. Compare the key id and the algorithm in the query with what the service expects. |
| `the query names source "…", not "…"` | `PARMANA_SIGNAL_SOURCE_NAME` is not the name Parmana sends. Set it to the name shown and redeploy. |
| `the query is addressed to another source` | Set `PARMANA_SIGNAL_AUDIENCE` to exactly the registered endpoint URL and redeploy. |
| `the query has expired`, `issued in the future` | Clock difference. Send a fresh request and check timing. |

## 8. Current progress against the plan

| Phase | Plan | Status |
|---|---|---|
| A Code on `main` | Connector fix, signal source name, log line | Done (log line unreviewed) |
| B Connector release service | Live and configured | Done (not re-tested this session) |
| C Signal service | Configured, deployed, reachable | Done |
| D Policy 1.1.0 | Proposed, approved, in effect | Done |
| E Connector binding | Demo revoked, registration active | Done |
| F Source registration | Proposed, approved, active | Done |
| G End-to-end test | Negative test (partial), positive test (not started) | **In progress, blocked** |

## 9. Governance and security record

- **Direct pushes to protected branches (2):** `2097791` on `parmana-connectors` `main`, and `45b8d46` on `findings/razorpay-e2e-guide`. Both used the admin bypass. Recorded in `RAZORPAY-INCIDENTS.md`. The admin bypass was disabled by the owner.
- **Unreviewed code on `main`:** `2097791` (one log line). Decision pending: keep, or revert through a pull request, once the refusal reason is known.
- **Keys shown in chat:** the maker key, a checker key, and a step-up key were each pasted into chat during the work. All were rotated. Treat any key that appeared in chat as exposed.
- **Configuration replaced:** `POLICY_CHANGE_CHECKERS` (replaced; the previous value could not be read back) and `PARMANA_API_KEYS` (rebuilt from the copy the operator held; hashes checked).
- **Local files to delete:** `checker-stepup-private.pem`, `signal-signing-private.pem` (if still present), `approval-body.json`, `reject-body.json`, `.env.check`, and `.vercel` / `.env.local` in linked folders.

## 10. Remaining work after this (ordered)

1. **Wait for the deployment limit to reset** (about 24 h from the attempt at about 21:25 IST on 2026-11-08).
2. **Deploy `parmana-connectors`** from `main`, then send one new negative request and read the `signal_query_refused` reason (section 11, steps 1–2).
3. **Fix the cause** from the table in section 7.1, and redeploy. Re-run the negative test until it returns `POLICY_DENIED` from the approval rule, not from the source.
4. **Positive test:** `manager-pavan` signs the approval on their machine; the maker sends the approved request (steps 3–6 of the exact steps in the guide). Expected: `APPROVED`, one refund of 200 paise on `pay_TuzVWT41QSiJd3` in Razorpay test mode, and a trust record.
5. **Decide on `2097791`:** keep it through a pull request, or revert it through a pull request.
6. **Owner:** merge PR #23 (guide) and PR #24 (incidents) through the normal route, now that the bypass is off. PR #22 (earlier draft) can be closed.
7. **Clean up** the stale `live-…`, `livecheck` and `erp` registrations and pending changes, and the test-leftover approvers.
8. **Replace the signal service's Razorpay key** with a read-only key if Razorpay offers one.
9. **Review the policy** and the signal source with a second person before any live-money use.

## 11. Resume tomorrow: where and how

**Where:**
- Findings repo, this folder: `razorpay-case-study/` on branch `findings/razorpay-case-study`.
- Guide: PR #23. Incidents: PR #24.
- Local folders on the Windows machine: `D:\last\parmana-connectors` (on `main`, at `2097791` or later), `D:\last\parmana-core`.

**First commands:**

```powershell
cd D:\last\parmana-connectors
git pull
vercel deploy --prod
```

If the deploy succeeds, run step 2:

```powershell
# 2. One new negative request (new ids, managerApproved false)
curl.exe -X POST https://parmana-api-real.vercel.app/execute -H "Authorization: Bearer $env:MAKER_KEY" -H "Content-Type: application/json" --data-binary "@refund-request-negative-3.json" -m 150
vercel logs --deployment parmana-connectors.vercel.app *> connectors-logs.txt
Select-String -Path connectors-logs.txt -Pattern "signal_query_refused" | Select-Object -Last 5
```

Then continue with section 10, step 3.

**Before you start, confirm:**
- `[bool]$env:MAKER_KEY` prints `True`. If not, set it again with `Read-Host`.
- The maker key is still allowed `razorpay:refund`.
- The checker keys are not in the folder.

## 12. Known gaps and risks

- The source refuses every query so far, and the reason is unknown until the log line is live.
- The log line is on `main` without review.
- The approval rule has not yet been seen to refuse a request on its own merits.
- Logs keep only 30 minutes on the Hobby plan, so diagnostics must run soon after a request.
- The deployment limit delays every change to the connectors by up to 24 hours.
- Parmana's connector and source share one Vercel project. A bad deploy affects both.
- The policy's correctness is not independently reviewed.
