# Relay `verifier` (attester)
Concordium Credential Verification Service (`concordium/credential-verification-service:0.1.0`), unmodified, used by `relay-service` for the **onboarding name + 18+ proof**. Implements D6a, TB5, TB13.

## 1. Configuration
| Setting | Value |
|---|---|
| Account | **Attester account** (`secrets/attester-account.export`), well-known, published in `attester.json`, funded with test CCD |
| gRPC | `GRPC_ENDPOINT` (testnet) |
| Ports | 8000 API, 8001 monitoring; Docker network only |

## 2. VPR template (`create-verification-request`)
```json
{
  "connectionId": "attest:<requestId>",
  "resourceId": "/attest/agent:ccd:testnet:9876/2a",
  "contextString": "Number One attestation: agent agent:ccd:testnet:9876/2a, owner ccd:testnet:acc:4VishwasXq",
  "publicInfo": { "app": "number-one", "requestId": "<requestId>" },
  "requestedClaims": [
    {
      "type": "identity",
      "source": ["accountCredential"],
      "issuers": ["did:ccd:testnet:idp:0", "did:ccd:testnet:idp:1", "did:ccd:testnet:idp:2", "did:ccd:testnet:idp:3"],
      "statements": [
        { "type": "RevealAttribute", "attributeTag": "firstName" },
        { "type": "RevealAttribute", "attributeTag": "lastName" },
        { "type": "AttributeInRange", "attributeTag": "dob", "lower": "18000101", "upper": "<today minus 18 years, YYYYMMDD>" }
      ]
    }
  ]
}
```
- `source: ["accountCredential"]` is essential. It ties the proof to account X through `credRegId` (TB5). **G1:** confirm in the M0 spike that the Browser Wallet answers this request type.
- **G1 is confirmed:** the Browser Wallet supports the Verify & Access flow (team check, Nov 2026). The M0 spike confirms the exact call and the name + age-range statements.
- `issuers` are built from `TRUSTED_IDPS`.

## 3. Endpoints used by `relay-service`
| Endpoint | When | Result used |
|---|---|---|
| `POST /verifiable-presentations/create-verification-request` | `POST /attestations/requests` | The VPR (anchors a VRA) |
| `POST /verifiable-presentations/verify` | `POST /attestations/requests/:id/presentation` | Result, audit record (hashed into `evidence.auditRecordHash`) and VAA transaction (`evidence.vaaTx`) |

## 4. Costs
Two `RegisterData` transactions per attestation (VRA + VAA). An attestation lasts 90 days, so with 3 demo users the cost is negligible.

## 5. Tests
- Create + verify round trip with a Browser Wallet test account (M0 G1 spike).
- A proof without `firstName` is rejected by the relay (`SCHEMA_INVALID`).
- A tampered VP fails verification.
