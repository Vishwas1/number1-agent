# Node bundle
One per user, on the user's own machine. Four services in one `docker compose` project: `ui`, `n1-agent`, `verifier`, `calendar-gateway`. Implements D7 and TB1, TB3.

## 1. `deploy/node/docker-compose.yml`
```yaml
services:
  ui:
    image: number-one/ui:0.1
    ports: ["127.0.0.1:${UI_PORT:-3000}:3000"]
    environment: [N1_API_URL=http://127.0.0.1:${N1_PORT:-7100}]
  n1-agent:
    image: number-one/n1-agent:0.1
    ports: ["127.0.0.1:${N1_PORT:-7100}:7100"]
    env_file: ${NODE_DIR:-.}/.env
    extra_hosts: ["host.docker.internal:host-gateway"]   # reach a relay running on the same machine (local testing)
    volumes:
      - ${NODE_DIR:-.}/data:/data                     # n1.db, keystore.json, audit/
      - ${NODE_DIR:-.}/secrets/node-service-account.export:/keys/service.export:ro
    depends_on: [calendar-gateway, verifier]
  calendar-gateway:
    image: number-one/calendar-gateway:0.1
    expose: ["7200"]
    env_file: ${NODE_DIR:-.}/.env
    volumes: ["${NODE_DIR:-.}/data/calendar:/data"]   # calendar.db (mock adapter)
  verifier:
    image: concordium/credential-verification-service:0.1.0
    expose: ["8000", "8001"]
    environment:
      - CREDENTIAL_VERIFICATION_SERVICE_NODE_GRPC_ENDPOINT=${GRPC_ENDPOINT}
      - CREDENTIAL_VERIFICATION_SERVICE_API_ADDRESS=0.0.0.0:8000
      - CREDENTIAL_VERIFICATION_SERVICE_MONITORING_ADDRESS=0.0.0.0:8001
      - CREDENTIAL_VERIFICATION_SERVICE_ACCOUNT=/keys/verifier_account.export
      - LOG_LEVEL=info
    volumes: ["${NODE_DIR:-.}/secrets/node-service-account.export:/keys/verifier_account.export:ro"]
```
- Host ports come from `UI_PORT` / `N1_PORT` (defaults 3000 / 7100), and all data paths from `NODE_DIR`, so several nodes can run on one machine ([local-dev.md](../local-dev.md)).
- `ui` and `n1-agent` bind to `127.0.0.1` only. `calendar-gateway` and `verifier` aren't published to the host at all.
- `n1-agent` connects **outbound only**: to the relay (WSS/HTTPS), the testnet gRPC endpoint, and the LLM provider.

## 2. `.env`
| Variable | Example | Used by |
|---|---|---|
| `NETWORK` | `testnet` | all |
| `UI_PORT` / `N1_PORT` | `3000` / `7100` | compose (host ports only; containers always listen on 3000 / 7100) |
| `NODE_DIR` | `.` | compose: folder holding this node's `.env`, `data/`, `secrets/`, relative to `deploy/node/` (e.g. `../local/kash`) |
| `GRPC_ENDPOINT` | `https://grpc.testnet.concordium.com:20000` | n1-agent, gateway, verifier |
| `CIS8004_CONTRACT` | `9876,0` | n1-agent, gateway |
| `RELAY_URL` / `RELAY_WS_URL` | `https://relay.number-one.dev` / `wss://relay.number-one.dev/ws` | n1-agent |
| `ATTESTER_PUBLIC_KEY` | `z6MkAttesterK1` | n1-agent (pinned, [attestation.md](../shared/attestation.md) §3) |
| `ATTESTER_ACCOUNT` | `3AttesterAcc…` | n1-agent (pinned) |
| `NODE_SERVICE_ACCOUNT` | `4NodeSvc…` | n1-agent, gateway (anchor sender check) |
| `KEYSTORE_PASSPHRASE` | (secret) | n1-agent |
| `LLM_MODE` | `openai` \| `scripted` | n1-agent |
| `OPENAI_API_KEY` | (secret, the owner's own key) | n1-agent |
| `LLM_MODEL` | `gpt-4o-mini` (any OpenAI chat model with JSON-schema output) | n1-agent |
| `OWNER_TZ` | `Europe/Copenhagen` | n1-agent, gateway |
| `GATEWAY_URL` / `VERIFIER_URL` | `http://calendar-gateway:7200` / `http://verifier:8000` | n1-agent |
| `N1_AGENT_URL` | `http://n1-agent:7101` | gateway (meeting lookups) |

## 3. Data on disk
| Path | Contents | Owner |
|---|---|---|
| `data/keystore.json` | Encrypted invocation, delegation and encryption keys | n1-agent |
| `data/n1.db` | SQLite (schema in [n1-agent.md](n1-agent.md) §4) | n1-agent |
| `data/calendar/calendar.db` | Mock calendar events | calendar-gateway |
| `secrets/node-service-account.export` | Node service account key (anchors only, never X) | verifier, n1-agent |

## 4. First run
1. `docker compose up -d`, then open `http://localhost:3000`.
2. The setup wizard runs ([ui.md](ui.md) §2, [flows.md](../flows.md) F1):
   - wallet link
   - choose crews
   - keys + card
   - register the N1
   - attestation
   - crew setup (Calendar: connect the mock calendar)
3. Setup state is kept in `n1.db`, so the wizard resumes where it stopped.
