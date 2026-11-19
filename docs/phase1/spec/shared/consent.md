# Commitments and consent
The **generic commitment protocol** that every crew uses for anything that commits the owner, plus approver consent with a **Verify & Access proof from the Browser Wallet**, bound to account X. Implements TB10, TB13 and A1. **POC deviation:** the target design (technical-solution D11) uses ID App consent on the phone. The POC uses the Browser Wallet at the desk instead ([poc-scope.md](../../poc-scope.md) §2). Crew-specific parts (the commitment content, consent text, re-check and execution) come from the crew module ([crew-modules.md](crew-modules.md) §3). The calendar crew's are in [crews/calendar.md](../crews/calendar.md) §7.

## 1. Roles
- **Requester:** the owner whose agent proposes the commitment (Kash). Their pick in their own UI is their intent, so they give no extra consent.
- **Approver:** the owner whose resource is committed (Vishwas). **Only the approver consents**, in their own local UI, with a proof from their **Browser Wallet** (account X).
- **The approver must be at their computer.** While they're away, the agents negotiate and the commitment waits as a tentative hold (D11). There's no phone approval: the ID App isn't used.

## 2. Commitment and hash
```
commitment     = crew-defined object, validated against the module's schema
commitmentHash = hex(sha256(JCS({ crew, commitmentId, requester, approver, commitment })))
```
`commitmentId` is a `urn:uuid`. `requester` and `approver` are agent IDs. The hash appears in the consent request's context, the `commitment.tentative` and `commitment.confirmed` messages, and the gateway's execute check.

## 3. Protocol (core messages)
| Type | Direction | Body | Authorization |
|---|---|---|---|
| `commitment.propose` | requester → approver | `{ crew, commitmentId, taskId, commitment, invocation }` | invocation of slip ② with the module's `proposeAction` (calendar: `calendar.meeting.propose`) |
| `commitment.tentative` | approver → requester | `{ commitmentId, commitmentHash, status: "awaiting_owner" }` | reply to a valid proposal |
| `commitment.confirmed` | approver → requester | `{ commitmentId, commitmentHash, consent: { auditRecordId, vaaTx } }` | approver's consent verified (§5) and executed |
| `commitment.declined` | approver → requester | `{ commitmentId, reason }` | — |

**Approver state machine** (core):
```
proposed ──validate + hold──► tentative ──owner approves (§4) + verified (§5) + recheck + execute──► confirmed
tentative ──owner declines / 24 h hold timeout / recheck fails──► declined (hold released)
```
On `commitment.propose`:
1. `verifyInvocation` with the module's `proposeAction`.
2. Validate against the module `schema` and `validate()`.
3. Check the module's `conflicts()` against active holds → `COMMITMENT_UNAVAILABLE`.
4. `recheck()` → `COMMITMENT_UNAVAILABLE`.
5. Store the hold (`commitments` table, `tentative`), compute `commitmentHash`, and reply `commitment.tentative`.
6. Show it in **Pending approvals** in the owner's UI (SSE `commitment`).

**Requester state machine** (core):
```
chosen ──commitment.propose──► proposed ──tentative──► tentative ──confirmed──► executing ──module.execute ok──► confirmed
any ──declined / error / 48 h timeout──► closed
```

## 4. Consent request (VPR, answered by the Browser Wallet)
1. The owner opens **Pending approvals**. The UI shows the module's `text(commitment, names)` and the short `commitmentHash`.
2. The owner clicks **Approve**, and the UI calls `POST /api/commitments/:id/approve/start`.
3. `n1-agent` creates the VPR through the node verifier ([node/verifier.md](../node/verifier.md)) and returns it to the UI.
4. The UI passes the VPR to the **Browser Wallet** (the same Verify & Access call as onboarding, [ui.md](../node/ui.md) §3), and the owner approves in the wallet.
5. The UI posts the VP to `POST /api/commitments/:id/approve/complete`.

Calendar example of the VPR:
```json
{
  "connectionId": "local:urn:uuid:1b7e5c90-4d2a-4f63-8c01-9e5a7d3b2f46",
  "resourceId": "/commitments/urn:uuid:1b7e5c90-4d2a-4f63-8c01-9e5a7d3b2f46",
  "contextString": "Number One approval: calendar commitment 8c41d2e7",
  "publicInfo": { "app": "number-one", "crew": "calendar", "commitmentId": "urn:uuid:1b7e5c90-4d2a-4f63-8c01-9e5a7d3b2f46", "commitmentHash": "8c41d2e7" },
  "requestedClaims": [
    {
      "type": "identity",
      "source": ["accountCredential"],
      "issuers": ["did:ccd:testnet:idp:0", "did:ccd:testnet:idp:1", "did:ccd:testnet:idp:2", "did:ccd:testnet:idp:3"],
      "statements": []
    }
  ]
}
```
- `source: ["accountCredential"]` binds the answer to account X through `credRegId` (§5 check 2). This closes G2.
- `statements: []` because nothing about the person needs proving; the binding to X is what matters. If the wallet requires at least one statement, use `RevealAttribute firstName` (check in the M1 spike).
- **No consent screen in the wallet.** The Browser Wallet doesn't render `consent:` contexts, so the human-readable text is shown **in the local UI** right before the wallet prompt, and the context string carries the commitment hash. That's acceptable because the local UI is the owner's own machine (TB1).
- **Hold timeout: 24 hours.** If the owner doesn't approve, the hold is released and the requester gets `commitment.declined`.

## 5. Verifying the consent
`n1-agent` calls the node verifier's `POST /verifiable-presentations/verify`. That returns the result, the audit record and the VAA transaction. The consent is accepted only if:
| # | Check | Reject with |
|---|---|---|
| 1 | Verifier result is success; VP context = the VPR context (`commitmentHash`, `resourceId` = the commitment) | `CONSENT_INVALID` / `CONSENT_COMMITMENT_MISMATCH` |
| 2 | The VP's account credential `credRegId` resolves (`accountByCredRegId`) to **this node's owner account X** | `CONSENT_NOT_OWNER` |
| 3 | The hold still exists, and the module's `recheck()` passes | `COMMITMENT_UNAVAILABLE` |

Then the core runs the module's `execute(commitment, { type: "consent", … })`, which calls the crew's gateway, and sends `commitment.confirmed`. Consent record:
```json
{
  "commitmentId": "urn:uuid:1b7e5c90-4d2a-4f63-8c01-9e5a7d3b2f46",
  "commitmentHash": "8c41d2e7",
  "account": "ccd:testnet:acc:4VishwasXq",
  "auditRecordId": "6e2f9b14-0a7c-4d58-9b3e-1c4d7a8f0e25",
  "auditRecord": {},
  "vaaTx": "d4c3b2a1",
  "verifiedAt": "2026-11-21T20:14:03Z"
}
```

## 6. Confirmation objects (for a gateway's execute action)
Every crew gateway checks one of these for actions marked `requiresConfirmation` ([crew-modules.md](crew-modules.md) §4a):
| Side | Object | Gateway checks |
|---|---|---|
| **Approver** | `{ "type": "consent", commitmentId, commitmentHash, account, auditRecord, vaaTx }` | `GET /internal/commitments/:id` → `role=approver`, same `commitmentHash`; the hash matches the execute params; the `auditRecord` context contains the hash and the result is success; `account` = own X; `sha256(auditRecord)` matches the VAA data; the VAA is from this node's service account |
| **Requester** | `{ "type": "approverConfirmed", envelope }`, the signed `commitment.confirmed` | the envelope verifies (`verifyAgent` + signature); `GET /internal/commitments/:id` → `role=requester`, `ownerChoiceAt` set, same `commitmentHash` |

## 7. Tests
- Approve → executed on both sides, and `commitment.confirmed` carries the VAA.
- Execute without a confirmation → `CONSENT_REQUIRED` (NT9).
- A consent for commitment A used for commitment B → `CONSENT_COMMITMENT_MISMATCH`.
- A VP from a different account (another Browser Wallet) → `CONSENT_NOT_OWNER`.
- No approval within 24 h → the hold is released, and the requester gets `commitment.declined`.
- A second proposal conflicting with a hold → `COMMITMENT_UNAVAILABLE`.
