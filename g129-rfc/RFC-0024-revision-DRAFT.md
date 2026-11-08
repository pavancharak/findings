# RFC-0024 (revision, DRAFT): Receiver-verified release for direct actions

Status: **Draft for review. Not decided. No code has been written.**
Created: 2026-11-08
Supersedes: the option A decision in `RFC-0024-Inbound-Agent-Authorization.md` (online check, agent identity)
Gap: G-129 (`VERIFICATION-GAPS.md`), GitHub issue pavancharak/parmana-core#421
Relates to: ADR-0013 (external connector, signed release), THREAT-MODEL.md (A8, assumption 9, E5)

---

## 1. Problem

Parmana governs an action only when it is routed through Parmana. Anyone who holds credentials to a
downstream system and calls it directly is outside the boundary. Their action is not checked and not
recorded (THREAT-MODEL A8; assumption 9 says this must be prevented by deployment, and Parmana cannot
enforce it in software).

The actor can be a person, a service, or an AI agent. The control must not depend on which.

## 2. Decision being proposed

The downstream system is the enforcement point. It accepts a governed action only when the request carries
a valid Parmana signed release for that exact action (E5 in THREAT-MODEL). Parmana's job is to make this
cheap to adopt, checkable by a third party, and visible when it is bypassed.

This replaces the earlier option A (the downstream system calls Parmana online before it acts, with an
agent-specific identity). Reasons:

- **Offline verification.** The receiver checks the signature with Parmana's public key. It does not
  depend on Parmana being reachable at the moment of the action.
- **One mechanism.** The release format and verifier already exist (ADR-0013, SDK helpers in TypeScript
  and Python). No second identity system is needed.
- **Actor-agnostic.** The release names the capability, target, parameters and approvals. It does not
  name a kind of caller.

## 3. Requirements

R1. **Receiver refuses without a valid release.** A governed system must verify, before acting:
    signature (Parmana's public key), audience (this endpoint), expiry, capability, and that the
    `businessTransactionId` was not executed before. It must refuse otherwise.

R2. **Language-neutral specification.** The release format and the verification steps are written as a
    spec, independent of any language or framework. Canonical serialization is defined exactly, so a
    receiver in any stack produces the same bytes.

R3. **Shared test vectors.** A fixed set of vectors that every receiver implementation must pass:
    valid release; bad signature; wrong audience; expired; replayed `businessTransactionId`; capability
    mismatch; body altered after signing; unknown key id; algorithm not in the allowed list.

R4. **Conformance is checked, and the result is signed.** The bypass probe (RFC-0024 decision 6, G-146)
    runs the vectors against a registered receiver and against a direct request with no release. It
    records a signed result. A receiver that does not pass, or is not checked, is reported as
    "not verified", never as covered.

R5. **Direct attempts become evidence.** A receiver that gets a request with no valid release records a
    signed refusal and reports it to Parmana. (Transport to be decided, see open questions.)

R6. **Algorithm agility.** The release names its signature algorithm and key id. The allowed set is
    configuration. Ed25519 is the initial value. ML-DSA-65 or a hybrid can be added without changing
    the format. (Relates to the post-quantum point; hybrid signing is G-119.)

R7. **No new identity system.** Caller authentication stays as it is. Nothing here issues agent
    credentials.

## 4. Non-goals

- Parmana does not stop a person or agent from using credentials it holds directly. It can only make that
  visible (R4, R5) and say so in its claims.
- Parmana does not verify a downstream system's internal logic.
- Parmana does not run customer code.

## 5. Residual risk (to be stated in the claim)

- A system that does not adopt R1 stays open. Coverage is per adopted receiver, not global.
- A receiver can be modified by whoever runs it (THREAT-MODEL A9). Conformance results prove the receiver
  passed the vectors at a time, not that it still does.
- Direct use of a credential that the receiver does not check remains invisible until R5 or R4 catches it.

Proposed claim wording: "Receivers that verify Parmana releases refuse governed actions without a valid
release. Receivers that are not checked by the probe are reported as not verified."

## 6. Acceptance criteria

- The spec and the vector set are published, versioned, and reviewed.
- At least two independent implementations pass every vector (TypeScript and Python already exist).
- The probe runs the vectors against one real pilot receiver and records a signed result.
- The claim wording in section 5 is added to CLAIMS.md, with its evidence.
- THREAT-MODEL A8 and assumption 9 are updated to reference this control.

## 7. Work plan (only after approval)

1. Write the spec (section 3 R2) and the vector set (R3). Generate vectors from the real server code.
2. Run the vectors against the existing TypeScript and Python verifiers. Fix any gap they show.
3. Extend the probe to run vectors and sign results (R4).
4. Receiver refusal reporting (R5) once transport is chosen.
5. Pilot receiver integration, live check, claim.

## 8. Open questions for the owner

1. Which receiver language comes after TypeScript and Python?
2. Which downstream system is the first pilot?
3. Transport for R5 refusal reports: signed webhook to Parmana, or a log feed that Parmana reads?
4. Who maintains the vector set when the release format changes?
5. Should R4 run on a schedule, on every deploy of the receiver, or both?

## 9. Decision record

| Date | Decision | By |
| ---- | -------- | -- |
| 2026-11-08 | Draft proposed | (pending review) |
| —    | Approved / changed / rejected | (owner) |
