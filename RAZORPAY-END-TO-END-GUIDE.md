# Razorpay refund: end-to-end guide

Status: **Draft, as of 2026-11-08.** Written from what was actually done in this work, not from the design alone.
Scope: the Razorpay refund path through Parmana, in Razorpay **test mode**.

---

## 1. Objective

Show that a refund on a Razorpay test payment can run only when both of these hold:

1. **Razorpay confirms the payment is refundable** for the requested amount. Parmana asks Razorpay itself, through a registered signal source. The caller cannot declare it.
2. **A named human has approved that payment and amount**, with a signed manager approval.

The refund then runs through the connectors release service, which performs the call to Razorpay and records the result.

This guide covers the setup, the approvals, what each step proves, and how to troubleshoot it.

---

## 2. Status at the time of writing

| Part | State | Record |
|---|---|---|
| Connector code (`razorpay:refund`) | On `main` in `parmana-connectors` | Squash commit `1eb59b5` (PR #61) |
| Connector service | Deployed; test-mode refund check passed earlier (Update 25 of FINDINGS.md) | Not re-run in this work |
| Demo registration of `razorpay:refund` | Revoked | Change `b9262dbb-9883-4ee8-8938-da8df94e539c`, approved by `mamta-checker` |
| Connector registration of `razorpay:refund` | Active; endpoint `https://parmana-connectors.vercel.app/api/execute` | Change `5d397f58-2e8a-4a10-9f61-75af40c3fad1`, approved by `mamta-checker` |
| Policy `razorpay-refund` | Version `1.1.0` in effect | Change `01b23fe0-874a-4e08-a7d9-43f40ea167a6`, approved by `mamta-checker` |
| Stale policy change on `razorpay-refund` 1.0.0 | Rejected | Change `cf85a4af-082a-424a-a646-31acc1cf8019`, rejected by `mamta-checker` |
| Signal service code (`/api/refund-payment`) | On `main`; source name now configurable | Commit `80ff107` |
| Signal service configuration | Set in production; deployed | Variables listed in section 5, phase C |
| Source registration `razorpay` | Active; endpoint `https://parmana-connectors.vercel.app/api/refund-payment`; key ID `signal-1` | Change `46bb4f5f-81c3-49ee-88cc-845fd1d49d4b`, approved by `mamta-checker` |
| End-to-end refund decision in test mode | **Not yet run** | See section 5, phase G |

---

## 3. How the parts fit together

```
Caller (agent or person)
   |  1. Asks for a refund: capability razorpay:refund, target pay_..., amount in paise
   v
Parmana API (parmana-api-real)
   |  2. Policy razorpay-refund 1.1.0 is evaluated
   |     - asks the registered source "razorpay" (signed query) whether payment.refundable holds
   |     - requires a signed manager approval for this payment and amount
   |  3. If approved: signed release is created
   v
Connector service (parmana-connectors /api/execute)
   |  4. Verifies the release signature, audience, expiry and single use
   |  5. Calls Razorpay refund API (test mode), with its own credential
   v
Razorpay (test mode)

Signal source (parmana-connectors /api/refund-payment)
   - answers step 2's question: reads the payment from Razorpay, signs the answer
```

Who holds which key:

| Key | Holder | Purpose |
|---|---|---|
| Maker bearer key | The person proposing changes | Proposes policy, registration and source changes |
| Checker bearer key | A different person | Approves or rejects changes |
| Checker step-up key (Ed25519 private key) | The same checker | Signs each approval or rejection, valid 120 seconds, single use |
| Parmana default public key | Public (fetched from `/keys/default`) | Lets the source and connector verify Parmana's signatures |
| Signal signing key (Ed25519) | Generated for the source service | Signs the source's answers. Its public half is registered with Parmana |
| Razorpay test key pair | Connector service and signal service | Calls Razorpay in test mode |

---

## 4. Prerequisites

- Windows PowerShell, in the folder of a local clone of `parmana-core` and `parmana-connectors`.
- `curl.exe` (not the PowerShell alias `curl`).
- Node.js 24 and npm. The commands below use `@parmana/sdk` (installed with `npm install @parmana/sdk`).
- OpenSSL (installed with Git for Windows) to generate the signal signing key.
- Vercel CLI, logged in, and linked to the `parmana-api-real` and `parmana-connectors` projects.
- A maker bearer key and a checker bearer key, each held by a different person.
- The checker's step-up private key file, `checker-stepup-private.pem`, present only while signing and deleted afterwards.
- A Razorpay **test-mode** key pair. Never use a live key for this procedure.

---

## 5. Steps, with commands

Every command below was run in this work, unless marked **not yet run**.

### Phase A: connector code on main

Already done: `parmana-connectors` `main` contains the Razorpay connector fix (`1eb59b5`) and the signal-source-name change (`80ff107`).

Check:

```powershell
cd D:\last\parmana-connectors
git pull
git log --oneline -3
```

Expected: `80ff107` is at the top, after the pull.

### Phase B: connector release service

Already configured and live. It was verified earlier with a test-mode refund (see FINDINGS.md, Update 25).

Its production variables (names only) are `PARMANA_PUBLIC_KEY`, `PARMANA_RELEASE_AUDIENCE`, `PARMANA_CONNECTOR`, `PARMANA_NONCE_STORE`, `PARMANA_ALLOW_LIVE_KEYS`, `PARMANA_DATABASE_URL` and `RAZORPAY_KEY_PAIR`.

Check the names:

```powershell
cd D:\last\parmana-connectors
vercel env ls production
```

### Phase C: signal service (the Razorpay refund-check source)

**C1. Generate the signing key pair, locally:**

```powershell
openssl genpkey -algorithm ed25519 -out signal-signing-private.pem
openssl pkey -in signal-signing-private.pem -pubout -out signal-signing-public.pem
```

**C2. Fetch Parmana's public key in PEM form:**

```powershell
(curl.exe -s https://parmana-api-real.vercel.app/keys/default | ConvertFrom-Json).pem | Set-Content -Encoding ascii parmana-default-public.pem
```

**C3. Add the production variables.** Use `Get-Content -Raw ... | vercel env add NAME production` so the file contents go in exactly as written. Each value is stored as a Secret.

```powershell
Set-Content -NoNewline -Encoding ascii signal-audience.txt "https://parmana-connectors.vercel.app/api/refund-payment"
Get-Content -Raw signal-audience.txt | vercel env add PARMANA_SIGNAL_AUDIENCE production
Get-Content -Raw parmana-default-public.pem | vercel env add PARMANA_SIGNAL_PARMANA_PUBLIC_KEY_PEM production
Get-Content -Raw signal-signing-private.pem | vercel env add PARMANA_SIGNAL_SIGNING_KEY_PEM production
Set-Content -NoNewline -Encoding ascii signal-source-name.txt "razorpay"
Get-Content -Raw signal-source-name.txt | vercel env add PARMANA_SIGNAL_SOURCE_NAME production
vercel env add RAZORPAY_CORE_READ_KEY production
```

`RAZORPAY_CORE_READ_KEY` is the only one typed at a prompt: choose **Secret**, then enter the Razorpay test key.

**C4. Delete the local key and temporary files, then deploy:**

```powershell
Remove-Item signal-signing-private.pem, signal-audience.txt, signal-source-name.txt -ErrorAction SilentlyContinue
vercel deploy --prod
```

Keep `signal-signing-public.pem`. Its contents go into the source registration (phase F).

**C5. Check that the route is configured.** A POST with an empty body must be refused by the service's own checks:

```powershell
curl.exe -s -w "`nHTTP %{http_code}`n" -X POST https://parmana-connectors.vercel.app/api/refund-payment -H "Content-Type: application/json" -d "{}"
```

Expected: `{"error":"the body has no query"}` with `HTTP 403`. If the response is a generic "not configured" error with HTTP 500, a variable is missing (see troubleshooting).

### Phase D: the policy `razorpay-refund` 1.1.0

**D1. The policy file** is `policies/razorpay-refund/1.1.0/policy.json` in `parmana-core`, on branch `feat/razorpay-payment-signal` (commit `36fe631`). It requires `paymentRefundable` (sourced from `razorpay`, for `amount`, valid 60 seconds) and `managerApproved` (an approval for the payment and amount).

**D2. Validate it** with the repo's validator (done in this work with `PolicyValidator`; the policy package tests passed, 554 tests).

**D3. Maker proposes it.** Build the proposal from the remote branch, so the local clone's state doesn't matter:

```powershell
$policy = (git show origin/feat/razorpay-payment-signal:policies/razorpay-refund/1.1.0/policy.json | Out-String) | ConvertFrom-Json
@{ proposedContent = $policy; reason = "Replace the demo razorpay-refund 1.0.0 with a version that requires Razorpay to confirm the payment is refundable, and a signed manager approval for that payment and amount." } | ConvertTo-Json -Depth 20 | Set-Content -Encoding utf8 policy-proposal.json
curl.exe -X POST "https://parmana-api-real.vercel.app/policies/razorpay-refund/1.0.0/pending-changes" -H "Authorization: Bearer $env:MAKER_KEY" -H "Content-Type: application/json" --data-binary "@policy-proposal.json"
```

The route's `version` (`1.0.0`) is the version being replaced. The body's `proposedContent` is the new version (`1.1.0`).

**D4. Checker approves** (see the approval procedure in section 6).

**D5. Confirm the version in effect:**

```powershell
curl.exe "https://parmana-api-real.vercel.app/policies/in-effect?capability=razorpay:refund" -H "Authorization: Bearer $env:MAKER_KEY"
```

Expected: `"policy":{"name":"razorpay-refund","version":"1.1.0",...}`.

### Phase E: the connector binding (`razorpay:refund`)

The binding is a registration, not a code change, because the built-in capability table is empty.

**E1. Revoke the demo registration** (maker proposes, a different checker approves):

```powershell
[System.IO.File]::WriteAllText("$PWD\revoke-demo.json", @'
{"action":"revoke","capability":"razorpay:refund","reason":"Demo registration is not in use. Replaced by the parmana-connectors registration."}
'@)
curl.exe -X POST https://parmana-api-real.vercel.app/external-connectors/changes -H "Authorization: Bearer $env:MAKER_KEY" -H "Content-Type: application/json" --data-binary "@revoke-demo.json"
```

**E2. Register the connector service** (maker proposes, a different checker approves). The body is `registration-proposal.json`, with `action: "register"`, `capability: "razorpay:refund"`, `endpointUrl: "https://parmana-connectors.vercel.app/api/execute"`, `policy: "razorpay-refund"`, `allowedParameters: ["amount"]`, `timeoutMs: 10000`.

```powershell
curl.exe -X POST https://parmana-api-real.vercel.app/external-connectors/changes -H "Authorization: Bearer $env:MAKER_KEY" -H "Content-Type: application/json" --data-binary "@registration-proposal.json"
```

Confirm with:

```powershell
curl.exe "https://parmana-api-real.vercel.app/external-connectors" -H "Authorization: Bearer $env:MAKER_KEY"
```

Expected for `razorpay:refund`: the connectors endpoint, status `active`; the demo entry, status `revoked`.

### Phase F: the signal source registration (`razorpay`)

**F1. Maker proposes** the source (`source-proposal.json`): `action: "register"`, `name: "razorpay"`, `endpointUrl: "https://parmana-connectors.vercel.app/api/refund-payment"`, `timeoutMs: 10000`, `publicKeyPem` (the contents of `signal-signing-public.pem`), `keyId: "signal-1"`.

```powershell
curl.exe -X POST https://parmana-api-real.vercel.app/business-signal-sources/changes -H "Authorization: Bearer $env:MAKER_KEY" -H "Content-Type: application/json" --data-binary "@source-proposal.json"
```

**F2. Checker approves** (section 6).

**F3. Confirm:**

```powershell
curl.exe "https://parmana-api-real.vercel.app/business-signal-sources" -H "Authorization: Bearer $env:MAKER_KEY"
```

Expected: the `razorpay` entry with status `active`. The list also contains many older `live-…` entries; they're test leftovers and can be ignored for this check.

### Phase G: end-to-end refund decision in test mode

**Not yet run.** The exact request format for a refund decision has to be confirmed against the agent guide in `docs/site/agents/` before running it. Plan:

1. Create a captured test-mode payment in Razorpay. Note its `pay_…` id and the amount in paise.
2. A maker proposes a refund for that payment, with a manager approval for that payment and amount. The approval is a separate signed step, by a different person.
3. Send the refund request to Parmana's `/execute` route, with the capability `razorpay:refund`, the target `pay_…`, and `parameters.amount` in paise.
4. Expected: Parmana evaluates `razorpay-refund` 1.1.0, asks the `razorpay` source, and then releases to the connector service, which refunds in test mode.
5. Check the Razorpay dashboard (test mode) for one refund with that amount and receipt, and check Parmana's record for the decision.

Record the Parmana decision ID, the Razorpay refund ID, and the time, in an update to FINDINGS.md.

---

## 6. The approval procedure (used for every change)

Run on the checker's own machine, with their own keys. Never on a shared machine.

```powershell
# 1. Set the checker key at the hidden prompt
$checkerSecure = Read-Host -AsSecureString "Checker key"
$env:CHECKER_KEY = [Runtime.InteropServices.Marshal]::PtrToStringAuto([Runtime.InteropServices.Marshal]::SecureStringToBSTR($checkerSecure))

# 2. Create the approval script (checker-stepup-private.pem must be in this folder)
npm install @parmana/sdk
@'
import { readFileSync } from "node:fs";
import { signPolicyChangeStepUp } from "@parmana/sdk";

const auth = signPolicyChangeStepUp({
  pendingPolicyChangeId: process.argv[2],
  action: "approve",
  privateKeyPem: readFileSync("checker-stepup-private.pem", "utf8"),
  keyId: "checker-stepup-1",
  ttlSeconds: 120,
});
console.log(JSON.stringify({ stepUpAuthorization: auth }));
'@ | Set-Content -Encoding utf8 approve-policy-step-up.mjs

# 3. Sign and approve in one go (the signature expires after 120 seconds)
node approve-policy-step-up.mjs CHANGE_ID | Set-Content -Encoding utf8 approval-body.json
curl.exe -X POST "https://parmana-api-real.vercel.app/<route>/CHANGE_ID/approve" -H "Authorization: Bearer $env:CHECKER_KEY" -H "Content-Type: application/json" --data-binary "@approval-body.json"

# 4. Delete the step-up key and temporary files
Remove-Item .\checker-stepup-private.pem, .\approval-body.json -ErrorAction SilentlyContinue
Remove-Item Env:CHECKER_KEY
```

The `<route>` is one of:
- `policies/pending-changes` (policy changes)
- `external-connectors/changes` (connector registrations and revocations)
- `business-signal-sources/changes` (source registrations)

Rejections use the same signing step, with `action: "reject"` and a `rejectionReason` in the body.

A successful approval returns `"status":"APPROVED"`, with `resolvedBy` set to the checker's caller ID.

---

## 7. Troubleshooting

| What you see | Cause | Fix |
|---|---|---|
| `{"error":"authentication required"}` (401) | The key in the variable is missing, empty, revoked, or not the one registered on the server | Check the variable with `$env:MAKER_KEY.Substring(0,4)`. Set it again with `Read-Host`. If it's a rotated key, make sure the new key is the one in use |
| Key starts with the old, revoked key's first characters | The clipboard or prompt still holds the old key | Generate a new key, copy it fresh, and check again |
| `POST : The term 'POST' is not recognized` | A URL was pasted on its own line, so PowerShell read it as a command | Always send requests with `curl.exe -X POST …` |
| `Malformed JSON body` | PowerShell removed the double quotes inside a single-quoted string passed to `curl.exe` | Write the JSON to a file with `[System.IO.File]::WriteAllText`, then send it with `--data-binary "@file.json"` |
| `curl: Failed to open <file>` | The file isn't in the current folder | Run `Test-Path .\<file>`. Create it in the current folder, or use the full path |
| `CHECKER_NOT_IN_SCOPE` | The checker is not listed for this policy or change type in `POLICY_CHANGE_CHECKERS` (or not on the right list) | Use a checker who is on the list. Otherwise, the operator adds them (see below) |
| `A PENDING_APPROVAL change already exists for policy … version …` (409) | Another change for the same policy and version is still open | List pending changes, then approve or reject the stale one before proposing again |
| `Capability 'razorpay:refund' already has an active external connector. Revoke it first.` (409) | An active registration already exists | Revoke it through a change, then register again |
| `ENOENT … checker-stepup-private.pem` | The checker's step-up key file isn't in the folder | The checker must put their own key file there. Don't use anyone else's key |
| Approval fails right after signing | The 120-second signature expired | Run the sign and approve steps back to back |
| `Cannot find module …approve-policy-step-up.mjs` | The script wasn't created in this folder | Run the script-creation command in section 6 first |
| `"error":"the body has no query"` (403) from the signal route | **Good.** The service is configured and refused an empty request | Nothing to fix |
| A generic "not configured" error (500) from the signal route | A required `PARMANA_SIGNAL_*` variable or `RAZORPAY_CORE_READ_KEY` is missing | Run `vercel env ls production` and compare with the list in phase C3. Then redeploy |
| `HTTP 000` from `curl.exe` | The sandbox or network can't reach the host | Run the request from your own machine |
| `vercel deploy` reports a rate limit | Vercel's build-rate limit | Wait for the reset (about 24 hours) and deploy again. Earlier PR checks failed the same way |
| `vercel env add` asks for a type | The variable's type | Choose Secret for credentials |
| A PR to `findings` main is refused | Branch protection is active | Open a PR from a branch. Direct pushes to `main` are blocked |
| Git says `main` is ahead or behind | A stale local ref | Run `git fetch --depth=50 origin "+refs/heads/main:refs/remotes/origin/main"` and compare the hashes |

### Adding a checker to the policy list (operator task)

`POLICY_CHANGE_CHECKERS` is a Secret in Vercel. It is a JSON object keyed by policy name, for example:

```json
{"razorpay-refund":["mamta-checker"]}
```

Replacing the whole variable removes checkers for every other policy. Keep the other entries, or confirm with the operator that they aren't needed. The current value can't be read back, because it's a Secret. Redeploy after the change, or the new value does not apply.

---

## 8. What this proves

- The policy `razorpay-refund` 1.1.0 is in effect for `razorpay:refund`. (Checked with `/policies/in-effect`.)
- The connector registration points at the connectors service, and the demo registration is revoked. (Checked with `/external-connectors`.)
- The `razorpay` source is registered and active, pointing at the signal service. (Checked with `/business-signal-sources`.)
- The signal service is deployed and configured. It answers an unsigned request with its own checks, not a generic configuration error. (Phase C5.)
- Every change in the list above went through a maker and a different checker, with a step-up signature. Each approval is recorded with `resolvedBy`.
- The stale policy change was rejected by a checker who was on the list, so it no longer blocks the 1.0.0 version.

## 9. What this does not prove

- **That a refund has ever run through the whole chain.** Phase G has not been run. Until it is, nothing shows that the signal query, the signed answer, the policy evaluation, the release and the Razorpay refund work together.
- **That the signal service's signed answers are accepted by Parmana.** The echo and signature checks run inside Parmana when a query is made, and no query has been made yet.
- **That the connector service's audience matches the registration.** Its `PARMANA_RELEASE_AUDIENCE` should equal `https://parmana-connectors.vercel.app/api/execute`. This was not checked in this work.
- **That live money is safe.** Everything here is test mode. `PARMANA_ALLOW_LIVE_KEYS` must stay off until a test-mode run passes.
- **That the policy is correct.** The policy shows which rules decided. It doesn't show the rules are right.
- **That other checkers are configured correctly.** Replacing `POLICY_CHANGE_CHECKERS` may have removed other policies' checker lists, and that is unverified.
- **That the stale `live-…` and `livecheck` registrations are harmless.** They're listed as active or revoked, and they point at example or test endpoints. They haven't been reviewed.

## 10. Security notes

- Keys were pasted into chat more than once during this work: the maker key, a checker key and a step-up key. All were rotated. Treat any key that appeared in chat as exposed.
- Keys must be typed at a hidden prompt (`Read-Host -AsSecureString`), not pasted into a command, and never printed.
- Delete local key and temporary files after each use: `checker-stepup-private.pem`, `approval-body.json`, `reject-body.json`, `signal-signing-private.pem`, `.env.check`.
- `vercel link` creates `.vercel` and `.env.local`, which hold a short-lived Vercel token. Delete them when they're no longer needed.
- The connector service and the signal service each hold a Razorpay key. The signal service's key should be read-only if Razorpay offers one. Here it's the test-mode key, which is more than the signal service needs.

## 11. Open items

1. Run phase G, the end-to-end test in test mode, and record the result.
2. Confirm the connector service's `PARMANA_RELEASE_AUDIENCE` matches its registered endpoint.
3. Decide whether `POLICY_CHANGE_CHECKERS` needs entries for other policies.
4. Review and clean up the stale `live-…`, `livecheck` and `erp` registrations and pending changes.
5. Replace the test-mode key used by the signal service with a read-only key, if Razorpay offers one.
6. Delete local clones' `.vercel` and `.env.local` once no longer needed.
