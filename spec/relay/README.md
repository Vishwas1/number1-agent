# Relay bundle
Company-hosted, and the only centralised component. Three parts: `relay-service`, an SQLite database (a file on a volume), and `verifier`. Implements D6a, D7, D9 and TB13, TB14.

## 1. `deploy/relay/docker-compose.yml`
```yaml
services:
  relay-service:
    image: number-one/relay-service:0.1
    ports: ["443:8443"]
    env_file: .env
    volumes:
      - ./data:/data                                   # relay.db
      - ./secrets/attester-signing.key:/keys/attester-signing.key:ro
      - ./tls:/tls:ro
    depends_on: [verifier]
  verifier:
    image: concordium/credential-verification-service:0.1.0
    expose: ["8000", "8001"]
    environment:
      - CREDENTIAL_VERIFICATION_SERVICE_NODE_GRPC_ENDPOINT=${GRPC_ENDPOINT}
      - CREDENTIAL_VERIFICATION_SERVICE_API_ADDRESS=0.0.0.0:8000
      - CREDENTIAL_VERIFICATION_SERVICE_MONITORING_ADDRESS=0.0.0.0:8001
      - CREDENTIAL_VERIFICATION_SERVICE_ACCOUNT=/keys/verifier_account.export
      - LOG_LEVEL=info
    volumes: ["./secrets/attester-account.export:/keys/verifier_account.export:ro"]
```
The database is SQLite in `data/relay.db`, opened by `relay-service` in WAL mode. It runs as a **single instance**: there's no horizontal scaling in the POC.

## 2. `.env`
| Variable | Example | Purpose |
|---|---|---|
| `PUBLIC_BASE_URL` | `https://relay.number-one.dev` | Card URIs, attester key ID |
| `GRPC_ENDPOINT` | `https://grpc.testnet.concordium.com:20000` | Chain reads |
| `CIS8004_CONTRACT` | `9876,0` | Registry |
| `DB_PATH` | `/data/relay.db` | SQLite |
| `VERIFIER_URL` | `http://verifier:8000` | Attester verifier |
| `ATTESTER_KEY_FILE` / `ATTESTER_KEY_ID` | `/keys/attester-signing.key` / `k1` | Ed25519 key that signs attestations |
| `ATTESTER_ACCOUNT` | `3AttesterAcc…` | Published in `attester.json`; the verifier's anchoring account |
| `TRUSTED_IDPS` | `0,1,2,3` | Issuers accepted in the attestation VPR |
| `ATTESTATION_TTL_DAYS` | `90` | Attestation lifetime |
| `QUEUE_TTL_DAYS` | `7` | Undelivered envelopes are dropped after this |
| `MAX_ENVELOPE_BYTES` | `65536` | Size limit |

## 3. Secrets
| File | What | Exposure if leaked |
|---|---|---|
| `attester-signing.key` | Signs attestations | **High**: false names. Rotate the key and re-pin it in every node's `.env`. Every issued attestation stays auditable via its VAA. |
| `attester-account.export` | Anchors VRA/VAA | Low: can only anchor data and spend CCD |
| TLS certificate | HTTPS/WSS | Standard |

The relay holds **no owner keys, no agent keys and no calendar data**.

## 4. Operations
- Back up `relay.db` daily. Losing it loses queued messages and the index. Nodes keep their verified copies, and owners can re-attest.
- Monitoring: `GET /health` (relay) and `:8001` (verifier).
