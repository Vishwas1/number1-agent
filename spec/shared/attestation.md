# Attestation (name + 18+)
Implements D6a, D9 and TB5, TB13, TB14 (G9).

## 1. Purpose
A Verify & Access proof can't be re-verified later. So the relay's **attester** verifies a fresh name + 18+ proof once, at onboarding, and signs an attestation. Peers check the attestation instead of the proof.

## 2. Format
```json
{
  "type": "NumberOneAttestationV1",
  "id": "urn:uuid:4c2e8f0a-0d7b-4a51-9a63-3f1b2b0f9e11",
  "issuer": "https://relay.number-one.dev/.well-known/attester.json#k1",
  "subject": {
    "agent": "agent:ccd:testnet:9876/2a",
    "owner": "ccd:testnet:acc:4VishwasXq"
  },
  "claims": {
    "firstName": "Vishwas",
    "lastName": "Anand",
    "ageOver18": true
  },
  "evidence": {
    "vraTx": "a1b2c3d4",
    "vaaTx": "e5f6a7b8",
    "auditRecordId": "0c8d6a9e-61f3-4a52-8b5e-0f6d2b7c1a90",
    "auditRecordHash": "3f9a6b2c"
  },
  "issued": "2026-11-20T09:05:00Z",
  "expires": "2027-02-18T09:05:00Z",
  "proof": {
    "type": "DataIntegrityProof",
    "cryptosuite": "eddsa-jcs-2022",
    "verificationMethod": "https://relay.number-one.dev/.well-known/attester.json#k1",
    "proofPurpose": "assertionMethod",
    "created": "2026-11-20T09:05:00Z",
    "proofValue": "z5AttesterSignature"
  }
}
```
| Field | Rule |
|---|---|
| `subject.agent`, `subject.owner` | The agent and account X named in the VPR context; checked against the chain at issue time |
| `claims` | Only what the VP revealed or proved. `ageOver18` is derived from the `dob` range statement. |
| `evidence.vaaTx` | The VAA anchored by the relay's verifier from the **attester account** |
| `evidence.auditRecordHash` | Hash of the verification audit record as anchored in the VAA (exact encoding per the Verifier Service; confirm in the M1 spike) |
| `expires` | `issued` + **90 days** (`ATTESTATION_TTL_DAYS`). After that, the owner re-attests. |

## 3. Attester key publication
- `GET <RELAY_PUBLIC_URL>/.well-known/attester.json` returns:
  ```json
  {
    "keys": [{ "id": "k1", "type": "Ed25519", "publicKeyMultibase": "z6MkAttesterK1" }],
    "attesterAccount": "ccd:testnet:acc:3AttesterAcc"
  }
  ```
- **Nodes pin both values in `.env`** (`ATTESTER_PUBLIC_KEY`, `ATTESTER_ACCOUNT`). They never trust the endpoint alone, because a compromised relay could swap the key. The endpoint is for discovery and auditors.

## 4. Issuing (relay side)
See [relay-service.md](../relay/relay-service.md) §3.3. Before signing, the attester checks:
1. the verifier returned `success` for the VP against the VPR it created
2. the VP's account credential `credRegId` resolves to an account (`getAccountInfo` by credRegId) that equals `subject.owner`
3. `verifyAgent(subject.agent, subject.owner)` passes
4. the VPR is unexpired, and its context names this agent and owner

## 5. Verifying (node side)
`verifyAttestation(att, agentView)`:
| # | Check | Reject with |
|---|---|---|
| 1 | Schema valid, `type` correct | `SCHEMA_INVALID` |
| 2 | `proof` verifies with the **pinned** attester key | `ATTESTATION_BAD_SIGNATURE` |
| 3 | `now < expires` | `ATTESTATION_EXPIRED` |
| 4 | `subject.agent` = the agent being checked; `subject.owner` = current on-chain owner | `ATTESTATION_SUBJECT_MISMATCH` |
| 5 | `vaaTx` is finalised, is a `RegisterData` from the pinned attester account, and its data matches `auditRecordHash` | `ATTESTATION_VAA_NOT_FOUND` |

- The result is cached per attestation ID. Checks 3 and 4 are re-run whenever `verifyAgent` runs.
- An NFT transfer changes the owner, so check 4 fails and the agent drops out of the directory automatically.

## 6. Tests
- A valid attestation passes.
- One signed by another key fails with `ATTESTATION_BAD_SIGNATURE` (NT11).
- An expired attestation fails with `ATTESTATION_EXPIRED` (NT11).
- An attestation for #42 presented by #99 fails with `ATTESTATION_SUBJECT_MISMATCH` (NT3).
- A wrong or missing VAA fails with `ATTESTATION_VAA_NOT_FOUND`.
