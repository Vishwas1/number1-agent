# Node `verifier`
Concordium Credential Verification Service (`concordium/credential-verification-service:0.1.0`), unmodified. It's used only for **the owner's own consent**. Implements D6b, TB10, TB13.

## 1. Purpose
Creates the consent request (anchoring a VRA) and verifies the Browser Wallet's answer (anchoring a VAA), so every confirmed commitment has on-chain evidence bound to the owner's account X.

## 2. Configuration
See the compose file in [README.md](README.md) §1.
| Setting | Value |
|---|---|
| Account | **Node service account** (`secrets/node-service-account.export`), funded with test CCD. **Never X.** |
| gRPC | `GRPC_ENDPOINT` (testnet) |
| Ports | 8000 API, 8001 monitoring; Docker network only |

## 3. Endpoints used by `n1-agent`
| Endpoint | When | Request | Result used |
|---|---|---|---|
| `POST /verifiable-presentations/create-verification-request` | The owner clicks **Approve** on a pending commitment | [consent.md](../shared/consent.md) §4 | VPR (`ConcordiumVerificationRequestV1`) → returned to the UI for the Browser Wallet |
| `POST /verifiable-presentations/verify` | The UI returns the Browser Wallet's VP | `{ auditRecordId: uuid, publicInfo: { crew, commitmentId }, presentation: VP, presentationRequest: VPR }` | Result, audit record and VAA transaction → the consent record |

- `connectionId` = `local:<commitmentId>` (there's no WalletConnect session; the UI talks to the Browser Wallet extension directly).
- `auditRecordId` = a fresh UUID per consent, stored in `consents`.
- The response fields for the audit record and the anchor transaction follow the Verify & Access guide. Confirm the exact names in the M1 spike and map them in `n1-agent`'s consent module.

## 4. Costs
Each consent anchors one VRA and one VAA from the service account. Budget about 2 `RegisterData` transactions per confirmed commitment. Top up the account manually on testnet.

## 5. Security notes
- **TB13:** it only ever verifies its own owner's consent. Peers rely on the approver's signed `commitment.confirmed` plus the VAA.
- If the service-account key leaks, the attacker can only anchor data and spend its CCD. It holds no owner authority.

## 6. Tests
- Health check on `:8001`.
- Create + verify round trip on testnet with a Browser Wallet test account, `source: accountCredential`, empty statements (M1 spike).
- A tampered VP is rejected (→ `CONSENT_INVALID`).
