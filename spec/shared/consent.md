# Commitments and consent
The **generic commitment protocol** that every crew uses for anything that commits the owner, plus approver consent through the ID App. Implements D11, TB10, TB13 and A1 (G2). Crew-specific parts (the commitment content, consent text, re-check and execution) come from the crew module ([crew-modules.md](crew-modules.md) §3). The calendar crew's are in [crews/calendar.md](../crews/calendar.md) §7.

## 1. Roles
- **Requester:** the owner whose agent proposes the commitment (Kash). Their pick in their own UI is their intent, so they give no extra consent.
- **Approver:** the owner whose resource is committed (Vishwas). **Only the approver consents**, in the ID App on their phone.
- The consent request goes **only** over the WalletConnect session that the owner paired with their own node at setup (G2 mitigation 1).

## 2. Commitment and hash
```
commitment     = crew-defined object, validated against the module's schema
commitmentHash = hex(sha256(JCS({ crew, commitmentId, requester, approver, commitment })))
```
`commitmentId` is a `urn:uuid`. `requester` and `approver` are agent IDs. The hash appears in the consent text (`Ref`), the `commitment.tentative` and `commitment.confirmed` messages, and the gateway's execute check.

## 3. Protocol (core messages)
| Type | Direction | Body | Authorization |
|---|---|---|---|
| `commitment.propose` | requester → approver | `{ crew, commitmentId, taskId, commitment, invocation }` | invocation of slip ② with the module's `proposeAction` (calendar: `calendar.meeting.propose`) |
| `commitment.tentative` | approver → requester | `{ commitmentId, commitmentHash, status: "awaiting_owner" }` | reply to a valid proposal |
| `commitment.confirmed` | approver → requester | `{ commitmentId, commitmentHash, consent: { auditRecordId, vaaTx } }` | approver's consent verified (§5) and executed |
| `commitment.declined` | approver → requester | `{ commitmentId, reason }` | — |

**Approver state machine** (core):
```
proposed ──validate + hold──► tentative ──consent verified + recheck + execute──► confirmed
tentative ──consent rejected / 24 h timeout / recheck fails──► declined (hold released)
```
On `commitment.propose`:
1. `verifyInvocation` with the module's `proposeAction`.
2. Validate against the module `schema` and `validate()`.
3. Check the module's `conflicts()` against active holds → `COMMITMENT_UNAVAILABLE`.
4. `recheck()` → `COMMITMENT_UNAVAILABLE`.
5. Store the hold (`commitments` table, `tentative`), compute `commitmentHash`, and reply `commitment.tentative`.
6. Start consent (§4).

**Requester state machine** (core):
```
chosen ──commitment.propose──► proposed ──tentative──► tentative ──confirmed──► executing ──module.execute ok──► confirmed
any ──declined / error / 48 h timeout──► closed
```

## 4. Consent request (VPR)
`n1-agent` calls the node verifier's `POST /verifiable-presentations/create-verification-request` ([node/verifier.md](../node/verifier.md)). Calendar example:
```json
{
  "connectionId": "<WalletConnect session topic paired at setup>",
  "resourceId": "/commitments/urn:uuid:1b7e5c90-4d2a-4f63-8c01-9e5a7d3b2f46",
  "contextString": "consent:markdown:\n## Confirm meeting\n\n- **With:** Kash Patel (Number One agent:ccd:testnet:9876/11)\n- **When:** Sat 28 Nov 2026, 20:00–21:00 (Europe/Copenhagen)\n- **What:** Drink\n\nRef: 8c41d2e7",
  "publicInfo": { "app": "number-one", "crew": "calendar", "commitmentId": "urn:uuid:1b7e5c90-4d2a-4f63-8c01-9e5a7d3b2f46" },
  "requestedClaims": [
    {
      "type": "identity",
      "source": ["identityCredential"],
      "issuers": ["did:ccd:testnet:idp:0", "did:ccd:testnet:idp:1", "did:ccd:testnet:idp:2", "did:ccd:testnet:idp:3"],
      "statements": [
        { "type": "RevealAttribute", "attributeTag": "firstName" },
        { "type": "RevealAttribute", "attributeTag": "lastName" }
      ]
    }
  ]
}
```
- **Text:**
  - `consent:markdown:` + the module's `text(commitment, names)` + `\n\nRef: <commitmentHash>`
  - names come from **attestations**, never from peer-supplied text
  - markdown only, no links
- **Statements:** reveal `firstName` and `lastName` (G2 mitigation 2). The human-in-loop doc lists "consent string + identity attributes" as supported; confirm in the M0 spike. If it isn't supported, send `"statements": []` and set `CONSENT_NAME_MATCH=false`.
- The request is sent with `request_verifiable_presentation_v1` over WalletConnect. **Timeout: 24 hours**, after which the consent fails with `CONSENT_EXPIRED`, the hold is released and the requester gets `commitment.declined`.

## 5. Verifying the consent
When the VP comes back, `n1-agent` calls `POST /verifiable-presentations/verify`. That returns the result, the audit record and the VAA transaction. The consent is accepted only if:
| # | Check | Reject with |
|---|---|---|
| 1 | Verifier result is success; VP context = the VPR context (`Ref` = `commitmentHash`, `resourceId` = the commitment) | `CONSENT_INVALID` / `CONSENT_COMMITMENT_MISMATCH` |
| 2 | It arrived on the paired session topic | `CONSENT_INVALID` |
| 3 | If name match is on: the revealed `firstName` and `lastName` equal the **owner's own attestation** claims | `CONSENT_INVALID` |
| 4 | The hold still exists, and the module's `recheck()` passes | `COMMITMENT_UNAVAILABLE` |

Then the core runs the module's `execute(commitment, { type: "consent", … })`, which calls the crew's gateway, and sends `commitment.confirmed`. Consent record:
```json
{
  "commitmentId": "urn:uuid:1b7e5c90-4d2a-4f63-8c01-9e5a7d3b2f46",
  "commitmentHash": "8c41d2e7",
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
| **Approver** | `{ "type": "consent", commitmentId, commitmentHash, auditRecord, vaaTx }` | `GET /internal/commitments/:id` → `role=approver`, same `commitmentHash`; the hash matches the execute params; the `auditRecord` context contains the hash and the result is success; `sha256(auditRecord)` matches the VAA data; the VAA is from this node's service account |
| **Requester** | `{ "type": "approverConfirmed", envelope }`, the signed `commitment.confirmed` | the envelope verifies (`verifyAgent` + signature); `GET /internal/commitments/:id` → `role=requester`, `ownerChoiceAt` set, same `commitmentHash` |

## 7. Tests
- Approve → executed on both sides, and `commitment.confirmed` carries the VAA.
- Execute without a confirmation → `CONSENT_REQUIRED` (NT9).
- A consent for commitment A used for commitment B → `CONSENT_COMMITMENT_MISMATCH`.
- A consent arriving on another session topic → `CONSENT_INVALID`.
- No answer within 24 h → `CONSENT_EXPIRED`, the hold is released, and the requester gets `commitment.declined`.
- A second proposal conflicting with a hold → `COMMITMENT_UNAVAILABLE`.
