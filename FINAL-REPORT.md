# Parmana security review: final report

Status: final for the scope below, updated for F7 (parmana-connect #49, open). Date of this report: 7 November 2026.
Repositories: `pavancharak/parmana-core`, `pavancharak/parmana-connectors`, `pavancharak/parmana-verifier`, `pavancharak/parmana-connect`.
Companion files: `FINDINGS.md` (chronological record of every update) in this repository.

## 1. Scope and method

- **Source review.** Read the production bootstrap, the execution gateway's verification and dispatch path, the release-control path, and the connectors' release check, ledger, configuration and server. The review was targeted, not line-by-line: `parmana-core` has 1,283 `.ts` files outside `node_modules`, and most were not read.
- **Test runs.** Ran each repository's suite on the Node version its own configuration requires. Live Supabase suites were run against a local Postgres 16 built from the repository's own migrations (`scripts/apply-all-migrations.sql`).
- **Central question.** Can any path execute a real external action without verified business authority, authorization and approved executable content being checked first? No such path was found in the code read.

## 2. Verified controls

- **Production defaults are fail-closed (core).** Policy governance and signing readiness are relaxed only when `NODE_ENV` is `test` or `development` and an explicit opt-in is set. The nonce store is in-memory only under `NODE_ENV=test`. The late-bound signal verifier throws until it is bound.
- **Gateway verification order (core, `packages/execution-gateway/src/ExecutionGateway.ts`).** Envelope and signature; executable content hash; policy hash and governance; signal hash and divergence; revocation; key scope; business conditions; authority grant. The nonce is consumed last, and only if every earlier check passed. `execute()` dispatches only when `valid` is true. A missing `signalsHash` is refused when the signal verifier is wired.
- **Release-control nonce (core, `createReleaseVerifier`).** The authorization's nonce is consumed once on the single release path. The signed `businessTransactionHash` covers the executable content, including `businessTransactionId`, so one authorization yields one transaction id.
- **Release signature (connectors, `src/service/release.ts`).** The Ed25519 signature covers the full canonical release. The SHA-512 commitment is accepted only above 4 KB. Audience, capability, issue and expiry times are checked, with a 60-second maximum lifetime.
- **Release ordering (connectors, `src/service/server.ts`).** Check, then claim in the ledger, then run the connector. A repeat of the same `businessTransactionId` returns the stored answer, or 409 while the first run is still in progress.
- **Configuration (connectors).** Requires an HTTPS audience and an Ed25519 public key. The nonce store defaults to Postgres.
- **Refund idempotency (connectors, Razorpay).** The refund receipt is the `businessTransactionId`, and an existing refund is checked before a new one is created.

## 3. Findings and status

| # | Finding | Severity | Status | Where |
|---|---|---|---|---|
| F1 | Incident runbook queried a table the live service no longer writes to; "no row" could be read as "never ran" | High (operational) | Fixed and merged | parmana-connectors #44 |
| F2 | The live Postgres release ledger (the production default) had no tests | High (test coverage) | Tests added and passing; merged | parmana-connectors #44 |
| F3 | Replay identity keyed only on `businessTransactionId` | Medium | Resolved: one authorization yields one transaction id (see section 2) | parmana-core `createReleaseVerifier` |
| F4 | Core replay integration test skipped without a database | Medium (test coverage) | Run against a local database; passes | parmana-core |
| F5 | Preliminary report's file count (866) did not match the repository | Low | Not reconciled. The count of `.ts` files excluding `node_modules` is 1,283 | n/a |
| F6 | Distrusted-root list, startup guard and tests did not match Node's bundled root store or eMudhra's statement | High (production startup) | Fixed and merged: list narrowed to G3, guard reports rather than refuses on zero matches, tests check invariants | parmana-core #495, #496, #497 |
| F7 | Connect consumed an authorization's nonce before checking the transaction hash, so a tampered request could block the legitimate one (denial of service for that authorization; no unauthorized execution) | Low to medium (availability) | Fixed in code and tested; PR open, awaiting DCO, CI and review | parmana-connect #49 |

Setup gaps, none affecting security:
- The `typescript` SDK build was missing before tests; `pretest` now builds it (merged, #495).
- The `re2` native module must be built for the running Node version (environment, not a repository change).
- The pre-commit hook now checks Node 24 and requires a `Signed-off-by` line (merged, #495).
- The Dockerfile keeps major-tag-plus-digest pins, which the repository's architecture test requires.

## 4. Distrusted roots: decisions and evidence

The distrust list is a security control, so each decision is recorded here.

**Current list (parmana-core `packages/api/src/bootstrap/distrustCompromisedRoots.ts`):** emSign ECC Root CA - G3 only, SHA-256 `86:A1:EC:BA:08:9C:4A:8D:3B:BE:27:34:C6:12:BA:34:1D:81:3E:04:3C:F9:E8:A8:62:CD:5C:57:A3:6B:BE:6B`.

| Root | SHA-256 | Decision | Basis |
|---|---|---|---|
| emSign ECC Root CA - G3 | `86:A1:EC:…:6B` | Distrusted | eMudhra's statement: the eight revoked intermediates sat under this root. Downloaded from `repository.emsign.com` and checked: subject and SHA-256 match. |
| emSign Root CA - G1 | `2D:3A:E8:…:67` (bundled by some Node builds) and `40:F6:AF:…:67` (Mozilla's NSS entry) | Not distrusted | eMudhra: "remain active and continue to be fully supported." Chrome stops trusting G1 certificates issued after 16 Nov 2026 (issuance timing, not a compromise). |
| emSign Root CA - C1 | `12:56:09:…:8F` | Not distrusted | Downloaded from `repository.emsign.com` and checked. eMudhra: Mozilla's removal is "voluntary and planned" and "does not pertain to this security incident." |
| emSign ECC Root CA - C3 | `BC:4D:80:…:B3` | Not distrusted | Same as C1. |

**Behaviour after the change.** The server distrusts only G3. On Node builds that bundle emSign Root CA - G1, the server now trusts G1 again, as it did before the distrust work. This was merged with the owner's confirmation (PR #497). The bundled roots depend on the exact Node build:
- Node v24.18.0 (Windows development machine) bundles G1 with SHA-256 `40:F6:…`, plus G3, C1 and C3.
- Node 24.22.1 (sandbox build) bundles G1 with SHA-256 `2D:3A:…` and not G3.
- Node 24.22.2 (CI and Docker image, from CI logs) bundles none of the four.

**Startup behaviour.** The server logs `distrustedRootsBundled` and `distrustedRootsRemoved` at startup. It refuses to start only if the bundled store is empty, which would mean TLS itself is broken. Zero listed roots is not an error: the distrust already holds.

**`40:F6:…` (G1).** Earlier in this review the value was described as a transcription error. That was wrong. It is the SHA-256 of an emSign Root CA - G1 certificate that Mozilla's NSS bug 1515457 lists as trusted, so it is a real root. It is not on the list because G1 is not compromised.

## 5. Test results

| Repository | Node | Database | Result |
|---|---|---|---|
| parmana-connectors | 22.23.4 | Local Postgres 16 | 207 of 207 pass |
| parmana-core | 24.22.1 | None (as CI runs) | 3,566 pass, 81 skipped, 0 failed |
| parmana-core | 24.22.1 | Local Postgres 16, live-suite opt-in | 3,641 of 3,641 pass (fresh clone, `typescript/dist` removed) |
| parmana-verifier | 24.22.1 | None needed | 68 of 68 pass |
| parmana-connect | 24.22.1 | Local Postgres 16 | 190 of 190 pass |

Notes:
- The Node 24.22.2 behaviour is known only from CI logs, not a local run; no 24.22.2 binary was available in the sandbox.
- CI on `main` after PR #497 merged: Docker, CodeQL, Python SDK and Scorecard passed; the main CI run was still in progress at the last check.

## 6. Sources

| Source | Status | How obtained |
|---|---|---|
| eMudhra, "Statement on Recent Certificate Security Incident" (updates 1 to 8, last updated 5 Nov 2026) | Primary | Full text supplied by the owner |
| eMudhra repository, emSign ECC Root CA - G3, C1, C3 | Primary | Downloaded by the owner from `repository.emsign.com` and checked |
| Mozilla bug 1515457 (root inclusion, four emSign roots and fingerprints) | Primary | Full text supplied by the owner |
| Mozilla NSS 3.131.1 release notes (bugs 2084101 and 2083709) | Primary | Downloaded by the owner from Firefox Source Docs and searched |
| Apple support page, compromised eMudhra intermediates (published 29 Oct 2026) | Primary | Downloaded by the owner and searched |
| Google, "Protecting Chrome and Android users..." (update of 4 Nov 2026) | Primary | Downloaded by the owner and searched |
| Secondary search results (vendor blogs, news sites) | Secondary, not relied on for decisions | Only where noted in `FINDINGS.md` |

## 7. Limitations

- The review is targeted. Most of `parmana-core`'s 1,283 `.ts` files were not read line by line.
- `parmana-verifier` and `parmana-connect` were tested, not reviewed for security.
- Connector adapters beyond the release path and Razorpay refund flow were not reviewed.
- The live Supabase suites ran against a local Postgres built from the repository's migrations, not against a Supabase project.
- The sandbox could not reach eMudhra, Apple, Chrome or Mozilla directly, so those sources were confirmed from the owner's downloads or pasted text.

## 8. Open items

- Chrome and Mozilla are still evaluating eMudhra's remaining roots. This affects this repository only if one of those roots is re-added to the list.
- Confirm the post-merge CI run on `main` for PR #497 finishes green.
- Decide whether the connector adapters and the verifier and connect packages need a security review.
- Merge parmana-connect #49 (F7) once DCO, CI and review are complete, then confirm F7 as closed.
